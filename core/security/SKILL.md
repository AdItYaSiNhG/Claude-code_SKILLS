---
name: security
description: >
  Use this skill for any security-related task: vulnerability scanning, secret detection,
  dependency auditing, OWASP checks, auth hardening, CSP/header audits, SBOM generation,
  IAM review, CVE triage, container image scanning, and pen-test report triage.
  Triggers: "security audit", "find vulnerabilities", "check for secrets", "harden auth",
  "scan dependencies", "OWASP", "CVE", "IAM review", "pen test", "SBOM", "supply chain".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Security Skill

## Why this skill exists

Security reviews done ad-hoc are inconsistent. This skill enforces a **systematic,
layered approach**: static analysis → secret detection → dependency audit →
auth/config hardening → runtime surface review → report generation.

Follow every section relevant to the user's request. Always produce a
**prioritised finding list** (Critical → High → Medium → Low) at the end.

---

## 1. Reconnaissance — understand the codebase first

Before running any tool, map the surface:

```bash
# language/framework detection
find . -maxdepth 3 -name "package.json" -o -name "requirements.txt" \
  -o -name "go.mod" -o -name "Cargo.toml" -o -name "pom.xml" \
  -o -name "build.gradle" | head -20

# entry points
find . -name "main.*" -o -name "index.*" -o -name "app.*" \
  -o -name "server.*" | grep -v node_modules | grep -v ".git" | head -20

# count lines per language
cloc . --quiet --exclude-dir=node_modules,.git,dist,build 2>/dev/null \
  || find . -name "*.py" -o -name "*.ts" -o -name "*.go" \
     | grep -v node_modules | wc -l
```

Document: languages, frameworks, entry points, external integrations (auth providers,
payment gateways, external APIs).

---

## 2. Secret & credential scanning

**Run first — secrets in source are critical severity by default.**

```bash
# Option A: gitleaks (preferred — git-aware)
gitleaks detect --source . --report-format json \
  --report-path /tmp/gitleaks-report.json 2>/dev/null \
  && cat /tmp/gitleaks-report.json | python3 -m json.tool | head -100

# Option B: truffleHog (deep git history scan)
trufflehog filesystem . --json 2>/dev/null | head -200

# Option C: grep fallback (always run as secondary pass)
grep -rn --include="*.env*" --include="*.json" --include="*.yaml" \
  --include="*.yml" --include="*.py" --include="*.ts" --include="*.js" \
  -E "(api_key|apikey|secret|password|passwd|token|private_key|aws_access|AKIA)['\"]?\s*[:=]\s*['\"]?[A-Za-z0-9/+]{8,}" \
  . 2>/dev/null | grep -v "node_modules\|.git\|test\|spec\|example\|sample"
```

**Check .env files committed to git:**
```bash
git log --all --full-history -- "**/.env" "**/.env.*" 2>/dev/null | head -30
git log --diff-filter=D --summary -- "**/.env" 2>/dev/null | head -20
```

**Findings format:**
- File path + line number
- Secret type (AWS key / GitHub token / DB password / etc.)
- Whether it's in current HEAD or only in git history
- Remediation: rotate immediately → remove from history (BFG/git-filter-repo)

---

## 3. Static Application Security Testing (SAST)

### JavaScript / TypeScript
```bash
# ESLint security plugin
npx eslint . --plugin security --rule 'security/detect-object-injection: error' \
  --rule 'security/detect-non-literal-regexp: warn' \
  --rule 'security/detect-possible-timing-attacks: warn' \
  --format json 2>/dev/null | python3 -m json.tool | head -200

# Semgrep (language-agnostic, highly accurate)
semgrep --config=p/javascript --config=p/typescript \
  --config=p/owasp-top-ten --json . 2>/dev/null | head -300
```

### Python
```bash
bandit -r . -f json -o /tmp/bandit-report.json \
  --exclude ./tests,./venv,./.venv 2>/dev/null \
  && cat /tmp/bandit-report.json | python3 -c \
  "import json,sys; d=json.load(sys.stdin); \
   [print(f\"{r['issue_severity']:8} {r['filename']}:{r['line_number']} {r['issue_text']}\") \
    for r in sorted(d['results'], key=lambda x: x['issue_severity'], reverse=True)]"
```

### Go
```bash
gosec -fmt json -out /tmp/gosec-report.json ./... 2>/dev/null \
  && cat /tmp/gosec-report.json | python3 -m json.tool | head -200
```

### Java / Kotlin
```bash
# SpotBugs with find-sec-bugs plugin (must be configured in build)
./mvnw spotbugs:check -Dspotbugs.failOnError=false 2>/dev/null | tail -50
# OR
./gradlew spotbugsMain 2>/dev/null | tail -50
```

### Universal (any language)
```bash
semgrep --config=p/owasp-top-ten --config=p/secrets \
  --config=p/xss --config=p/sql-injection \
  --json . 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
results = data.get('results', [])
for r in sorted(results, key=lambda x: x.get('extra',{}).get('severity',''), reverse=True):
    print(f\"{r.get('extra',{}).get('severity','?'):8} {r['path']}:{r['start']['line']} {r['extra'].get('message','')[:80]}\")
print(f'\nTotal: {len(results)} findings')
"
```

---

## 4. Dependency vulnerability audit

### Node.js
```bash
npm audit --json 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
vulns = data.get('vulnerabilities', {})
for name, v in vulns.items():
    sev = v.get('severity','?')
    via = ', '.join(str(x.get('title','?')) if isinstance(x, dict) else str(x) for x in v.get('via',[])[:2])
    print(f'{sev:8} {name}: {via}')
print(f\"\nTotal: {data.get('metadata',{}).get('vulnerabilities',{})} \")
"

# Check for outdated packages too
npm outdated 2>/dev/null | head -30
```

### Python
```bash
pip-audit --format json 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for dep in data.get('dependencies',[]):
    for vuln in dep.get('vulns',[]):
        print(f\"{dep['name']}=={dep['version']}: {vuln['id']} - {vuln['description'][:80]}\")
"
# fallback
safety check --json 2>/dev/null | head -100
```

### Go
```bash
govulncheck ./... 2>/dev/null | head -100
```

### Java
```bash
# OWASP Dependency Check
dependency-check --project "audit" --scan . \
  --format JSON --out /tmp/dc-report 2>/dev/null \
  && cat /tmp/dc-report/dependency-check-report.json \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
for dep in d.get('dependencies',[]):
    for v in dep.get('vulnerabilities',[]):
        print(f\"{v.get('severity','?'):8} {dep.get('fileName','?')}: {v.get('name','?')} - {v.get('description','')[:60]}\")
" 2>/dev/null | head -100
```

### Rust
```bash
cargo audit 2>/dev/null | head -100
```

---

## 5. OWASP Top 10 manual checklist

For each item, grep the codebase for patterns and inspect:

```bash
# A01 Broken Access Control
# Look for missing auth middleware
grep -rn "router\.\(get\|post\|put\|delete\|patch\)" --include="*.ts" --include="*.js" . \
  | grep -v "auth\|middleware\|protect\|guard\|verify\|jwt\|session" \
  | grep -v "node_modules\|test\|spec" | head -30

# A02 Cryptographic Failures — weak algorithms
grep -rn "md5\|sha1\b\|DES\|RC4\|ECB\|createCipher\b" \
  --include="*.py" --include="*.ts" --include="*.js" --include="*.go" \
  . | grep -v "node_modules\|test\|comment" | head -20

# A03 Injection — raw query construction
grep -rn "query\s*=\s*[\"'].*\+\|execute\s*(\s*f[\"']\|cursor\.execute.*%\|db\.raw\b" \
  --include="*.py" --include="*.ts" --include="*.js" --include="*.go" \
  . | grep -v "node_modules\|test" | head -20

# A05 Security Misconfiguration — debug/dev flags in prod config
grep -rn "DEBUG\s*=\s*True\|debug:\s*true\|NODE_ENV.*development" \
  --include="*.py" --include="*.ts" --include="*.js" --include="*.yaml" \
  --include="*.env*" . | grep -v "node_modules\|test\|example" | head -20

# A07 Identification & Authentication Failures — weak session config
grep -rn "secret.*['\"].\{1,15\}['\"]" \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test" | head -20

# A09 Security Logging & Monitoring Failures — sensitive data in logs
grep -rn "console\.log\|print\|logger\.\(info\|debug\)" \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -i "password\|token\|key\|secret\|credit" \
  | grep -v "node_modules\|test" | head -20
```

---

## 6. Auth & session hardening review

```bash
# JWT configuration checks
grep -rn "jwt\|jsonwebtoken\|jose\|python-jose" \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test" | head -20

# Look for: algorithm=none, HS256 with weak secrets, missing expiry, no refresh rotation
grep -rn "algorithm.*none\|alg.*none\|verify.*false\|expiresIn.*['\"]" \
  . | grep -v node_modules | head -20

# Session cookies — check for secure/httponly/samesite
grep -rn "cookie\|session" --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test" \
  | grep -iv "httponly\|secure\|samesite" | head -20

# Password hashing — detect plain storage or weak hashing
grep -rn "password\|passwd" --include="*.py" --include="*.ts" --include="*.js" \
  . | grep -v "node_modules\|test\|comment\|#" \
  | grep -iv "bcrypt\|argon2\|scrypt\|pbkdf2\|hash" | head -20
```

---

## 7. HTTP security headers audit

```bash
# Check server/framework config for security headers
grep -rn "helmet\|Content-Security-Policy\|X-Frame-Options\|HSTS\|X-Content-Type" \
  --include="*.ts" --include="*.js" --include="*.py" --include="*.go" \
  . | grep -v "node_modules\|test" | head -30

# Express.js — confirm helmet is used
grep -rn "app\.use(helmet" . | grep -v node_modules | head -5

# Django — check SECURE_* settings
grep -rn "SECURE_\|X_FRAME_OPTIONS\|CSRF_COOKIE_SECURE" \
  --include="*.py" . | head -20
```

**Required headers checklist:**
- `Content-Security-Policy` — prevents XSS
- `Strict-Transport-Security` — enforces HTTPS
- `X-Frame-Options: DENY` — prevents clickjacking
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy`

---

## 8. Container & infrastructure security

```bash
# Dockerfile checks
find . -name "Dockerfile*" | while read f; do
  echo "=== $f ==="
  # running as root?
  grep -n "USER" "$f" | head -5
  # latest tags (non-pinned)
  grep -n "FROM" "$f" | grep -v "@sha256:" | head -5
  # secrets in ENV/ARG
  grep -n "ENV\|ARG" "$f" | grep -i "secret\|key\|password\|token" | head -5
done

# docker-compose secrets exposure
find . -name "docker-compose*.yml" -o -name "docker-compose*.yaml" \
  | xargs grep -n "password\|secret\|key" 2>/dev/null \
  | grep -v "#" | head -20

# Kubernetes — privileged containers, host mounts
find . -name "*.yaml" -o -name "*.yml" \
  | xargs grep -ln "privileged:\|hostNetwork:\|hostPID:" 2>/dev/null | head -10
```

---

## 9. Supply chain & SBOM

```bash
# Generate SBOM (Software Bill of Materials)
# Node.js
npx @cyclonedx/cyclonedx-npm --output-file /tmp/sbom-node.json 2>/dev/null

# Python
pip install cyclonedx-bom 2>/dev/null
cyclonedx-py -e -o /tmp/sbom-python.json 2>/dev/null

# Check for typosquatting risks in package.json
cat package.json 2>/dev/null \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
print(f'Total packages: {len(deps)}')
# Flag very short package names (typosquatting risk)
short = [k for k in deps if len(k) <= 3]
if short: print(f'Short names (review): {short}')
"
```

---

## 10. IAM & least privilege review

```bash
# AWS IAM — look for wildcard policies in IaC
grep -rn '"Action"\s*:\s*"\*"\|Action.*\*\|Effect.*Allow.*\*' \
  --include="*.json" --include="*.yaml" --include="*.yml" --include="*.tf" \
  . | grep -v node_modules | head -20

# Terraform IAM permissive policies
grep -rn "actions\s*=\s*\[\"\\*\"\]\|resources\s*=\s*\[\"\\*\"\]" \
  --include="*.tf" . | head -20

# Hardcoded AWS account IDs / ARNs
grep -rn "arn:aws:" --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test\|iam.amazonaws.com" | head -20
```

---

## 11. Finding report generation

After all scans, produce a structured report:

```
## Security Audit Report
**Date:** $(date)
**Scope:** [repo/path]
**Tools run:** [list tools that succeeded]

### Critical (fix immediately — production risk)
| # | Finding | File | Line | Remediation |
|---|---------|------|------|-------------|
| 1 | Hardcoded AWS secret key | src/config.ts | 42 | Rotate key, use env vars / secrets manager |

### High (fix before next release)
...

### Medium (fix within sprint)
...

### Low / Informational
...

### Summary
- Secrets found: N
- Vulnerable dependencies: N (critical: X, high: Y)
- SAST findings: N
- Auth issues: N
- Header misconfigurations: N

### Immediate actions
1. ...
2. ...
```

---

## Severity classification

| Severity | Definition | SLA |
|----------|-----------|-----|
| Critical | Exposed secret / unauthenticated RCE / SQL injection in prod | Fix now, deploy hotfix |
| High | Auth bypass / privilege escalation / known CVE (CVSS ≥7) | Fix this sprint |
| Medium | Missing headers / weak crypto / info disclosure | Fix next sprint |
| Low | Code quality / defence-in-depth / CVSS < 4 | Backlog |

---

## Common remediation snippets

### Secrets management (Node.js)
```typescript
// BAD
const apiKey = "sk-abc123...";

// GOOD — use environment variables validated at startup
import { z } from "zod";
const env = z.object({
  API_KEY: z.string().min(20),
  DATABASE_URL: z.string().url(),
}).parse(process.env);
```

### Parameterised queries (Python/SQLAlchemy)
```python
# BAD
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# GOOD
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

### Helmet.js (Express)
```typescript
import helmet from "helmet";
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
}));
```

### Password hashing (Node.js)
```typescript
import bcrypt from "bcryptjs";
const SALT_ROUNDS = 12;
const hash = await bcrypt.hash(password, SALT_ROUNDS);
const valid = await bcrypt.compare(input, hash);
```
