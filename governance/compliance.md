---
name: governance-compliance
description: Claude Code acceptable-use policy enforcement, data handling rules (PII/secrets in prompts), audit trail generation, and FOSS license compliance. Use for policy design, automated policy checks, audit log queries, and open-source license scanning.
---

The user is designing or enforcing AI usage compliance policies. Apply the relevant section.

## Acceptable-Use Policy Framework

### Policy categories to define

```markdown
# Claude Code Acceptable Use Policy v1.0

## Permitted uses
- Code generation, review, and refactoring within company projects
- Documentation generation from existing codebases
- Debugging assistance on internal systems
- Test case generation
- Architecture discussion and design exploration

## Restricted uses (require explicit approval)
- Generating code that processes or stores regulated data (PII, PHI, PCI)
- Using customer data samples in prompts
- Generating code for external publication or open-source contribution
- Using Claude Code on systems under active security incident

## Prohibited uses
- Pasting production credentials, API keys, or passwords into prompts
- Including real customer PII (names, emails, IDs) in prompts
- Using Claude Code to circumvent code review processes
- Generating code designed to bypass security controls
- Processing data subject to export control (ITAR/EAR) in shared environments
```

### Enforcement tiers

```
Tier 1 — Awareness (low risk)
  Action: Log the event, include in monthly usage report
  Examples: High token usage, unusual session length

Tier 2 — Warning (medium risk)
  Action: Slack DM to user + manager notification
  Examples: Potential PII detected in prompt, high-cost session

Tier 3 — Block + Alert (high risk)
  Action: Reject the API call, page security team
  Examples: Credentials detected in prompt, prohibited data category
```

## PII/Secret Detection in Prompts

### Pre-prompt scanner (server-side API proxy)
```python
import re
from dataclasses import dataclass

@dataclass
class ScanResult:
    is_clean: bool
    violations: list[dict]

# Detection patterns
PATTERNS = {
    "aws_key":      (r"AKIA[0-9A-Z]{16}", "HIGH"),
    "generic_secret": (r"(?i)(secret|password|passwd|token|api_key)\s*[:=]\s*\S+", "HIGH"),
    "email":        (r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}", "MEDIUM"),
    "phone_in":     (r"\+?91[-\s]?\d{10}", "MEDIUM"),         # Indian phone
    "pan_card":     (r"[A-Z]{5}[0-9]{4}[A-Z]", "MEDIUM"),    # Indian PAN
    "aadhaar":      (r"\b[2-9]{1}[0-9]{3}[-\s]?[0-9]{4}[-\s]?[0-9]{4}\b", "HIGH"),
    "credit_card":  (r"\b(?:\d{4}[-\s]?){3}\d{4}\b", "HIGH"),
    "private_key":  (r"-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----", "CRITICAL"),
    "jwt":          (r"eyJ[a-zA-Z0-9_-]{10,}\.eyJ[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}", "HIGH"),
}

def scan_prompt(text: str) -> ScanResult:
    violations = []
    for category, (pattern, severity) in PATTERNS.items():
        matches = re.findall(pattern, text)
        if matches:
            violations.append({
                "category": category,
                "severity": severity,
                "count": len(matches),
                "sample": matches[0][:20] + "..." if len(matches[0]) > 20 else matches[0],
            })
    return ScanResult(is_clean=len(violations) == 0, violations=violations)
```

### API proxy enforcement
```python
# Intercept all Claude API calls, scan before forwarding
async def proxy_handler(request: Request) -> Response:
    body = await request.json()
    
    # Extract all text content from messages
    all_text = " ".join(
        item["text"]
        for msg in body.get("messages", [])
        for item in (msg["content"] if isinstance(msg["content"], list) else [{"text": msg["content"]}])
        if item.get("type") == "text"
    )
    
    scan = scan_prompt(all_text)
    
    if not scan.is_clean:
        critical = [v for v in scan.violations if v["severity"] == "CRITICAL"]
        high = [v for v in scan.violations if v["severity"] == "HIGH"]
        
        if critical or high:
            await audit_log(
                user=request.headers.get("X-User-ID"),
                action="prompt_blocked",
                violations=scan.violations,
            )
            return JSONResponse(
                status_code=400,
                content={"error": "Prompt blocked: potential sensitive data detected", 
                         "violations": [v["category"] for v in scan.violations]}
            )
        else:
            await audit_log(user=..., action="prompt_warned", violations=scan.violations)
    
    # Forward to Anthropic
    return await forward_to_anthropic(body)
```

## FOSS License Compliance

### License scanning (CI-integrated)
```bash
# FOSS license audit — run before any open-source contribution or acquisition
npx license-checker --production --failOn "GPL-2.0;GPL-3.0;AGPL-3.0;LGPL-2.0;LGPL-2.1;LGPL-3.0"

# Python
pip-licenses --format=csv --output-file=licenses.csv
# Flag: GPL, AGPL, EUPL

# Go
go-licenses check ./... --allowed_licenses=MIT,Apache-2.0,BSD-2-Clause,BSD-3-Clause,ISC

# Java (Maven)
mvn license:check -Dlicense.failIfWarning=true
```

### License risk matrix
| License | Commercial SaaS | Internal tools | Distribution |
|---|---|---|---|
| MIT, BSD, Apache-2.0, ISC | ✅ Safe | ✅ Safe | ✅ Safe |
| LGPL (dynamic link) | ✅ Generally safe | ✅ Safe | ⚠️ Review |
| GPL-2.0 / GPL-3.0 | ❌ Block | ⚠️ Review | ❌ Block |
| AGPL-3.0 | ❌ Block | ❌ Block | ❌ Block |
| Commercial / Proprietary | ⚠️ Check terms | ⚠️ Check terms | ❌ Block |
| Creative Commons (code) | ❌ Not for code | ❌ Not for code | ❌ Block |

## Audit Trail

### Audit log schema
```sql
CREATE TABLE ai_audit_log (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id      TEXT NOT NULL,
    team         TEXT,
    action       TEXT NOT NULL,   -- 'prompt_sent' | 'prompt_blocked' | 'prompt_warned' | 'budget_exceeded'
    severity     TEXT,            -- 'INFO' | 'WARN' | 'HIGH' | 'CRITICAL'
    model        TEXT,
    input_tokens INTEGER,
    output_tokens INTEGER,
    violations   JSONB,           -- [{category, severity, sample}]
    session_id   TEXT,
    request_id   TEXT,
    metadata     JSONB            -- additional context
);

-- Retention: 2 years minimum for compliance
CREATE INDEX idx_audit_user_date ON ai_audit_log (user_id, occurred_at DESC);
CREATE INDEX idx_audit_action ON ai_audit_log (action, occurred_at DESC);
```

### Quarterly compliance report query
```sql
SELECT
    DATE_TRUNC('month', occurred_at)       AS month,
    action,
    severity,
    COUNT(*)                               AS event_count,
    COUNT(DISTINCT user_id)                AS unique_users,
    ARRAY_AGG(DISTINCT violations->>0)     AS violation_types
FROM ai_audit_log
WHERE occurred_at >= NOW() - INTERVAL '3 months'
  AND action != 'prompt_sent'             -- exclude routine events
GROUP BY 1, 2, 3
ORDER BY 1 DESC, event_count DESC;
```

## Output Format

1. For policy design: provide the full policy document in Markdown, ready to commit
2. For scanner implementation: show the complete scanner + proxy code
3. For license audit: show the scan command + risk matrix for the user's dependency list
4. For audit reporting: show the SQL query + sample output table
