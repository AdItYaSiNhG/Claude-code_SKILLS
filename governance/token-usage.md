---
name: token-usage-governance
description: >
  Track, report, and optimise Claude Code token usage and costs across sessions.
  Generates cost-per-session reports, monitors model tier selection, detects usage
  anomalies, enforces prompt compression, and calculates ROI on AI-assisted development.
  Use at the end of a work session, for billing review, or when costs are unexpectedly high.
  Triggers: "token usage", "how much did this cost", "session cost", "usage report",
  "AI spend", "token report", "cost report", "5 hour session", "billing", "ROI",
  "token optimisation", "reduce costs", "model selection".
compatibility: "Claude Code (claude.ai, Claude Desktop)"
version: "1.0.0"
---

# Token Usage & Governance Skill

## Why this skill exists

Claude Code sessions can consume thousands of tokens without developers noticing
until the bill arrives. This skill gives you **visibility, control, and
optimisation levers** for AI-assisted development costs — per session, per
feature, per team member.

---

## 1. Session cost estimator

Run this at the end of a work session to estimate what was spent:

```python
#!/usr/bin/env python3
# scripts/session-cost-report.py
# Usage: python scripts/session-cost-report.py --hours 5

import argparse
from datetime import datetime, timedelta
from dataclasses import dataclass

# Current pricing (USD per 1M tokens) — update as pricing changes
MODEL_PRICING = {
    "claude-opus-4-6":              {"input": 15.00, "output": 75.00},
    "claude-sonnet-4-20250514":     {"input":  3.00, "output": 15.00},
    "claude-haiku-4-5-20251001":    {"input":  0.25, "output":  1.25},
}

@dataclass
class SessionEstimate:
    model:         str
    hours:         float
    tasks:         int
    input_tokens:  int
    output_tokens: int

    @property
    def cost_usd(self) -> float:
        p = MODEL_PRICING[self.model]
        return (self.input_tokens * p["input"] + self.output_tokens * p["output"]) / 1_000_000

# Typical token usage patterns per task type
TASK_ESTIMATES = {
    "code_generation":   {"input": 3_000,  "output": 1_500},
    "code_review":       {"input": 5_000,  "output": 800},
    "debugging":         {"input": 4_000,  "output": 600},
    "refactoring":       {"input": 6_000,  "output": 2_000},
    "documentation":     {"input": 2_000,  "output": 1_000},
    "test_generation":   {"input": 4_000,  "output": 1_800},
    "architecture":      {"input": 3_000,  "output": 1_200},
}

def estimate_session(hours: float, model: str = "claude-sonnet-4-20250514") -> dict:
    """Estimate cost for a development session."""
    tasks_per_hour = 8   # average Claude Code tasks per hour
    tasks = int(hours * tasks_per_hour)

    # Assume mixed task types
    total_input = total_output = 0
    for i in range(tasks):
        task_type = list(TASK_ESTIMATES.keys())[i % len(TASK_ESTIMATES)]
        total_input  += TASK_ESTIMATES[task_type]["input"]
        total_output += TASK_ESTIMATES[task_type]["output"]

    est = SessionEstimate(model=model, hours=hours, tasks=tasks,
                          input_tokens=total_input, output_tokens=total_output)

    return {
        "session_duration_hours": hours,
        "estimated_tasks":        tasks,
        "model":                  model,
        "input_tokens":           total_input,
        "output_tokens":          total_output,
        "total_tokens":           total_input + total_output,
        "estimated_cost_usd":     round(est.cost_usd, 4),
        "cost_per_hour":          round(est.cost_usd / hours, 4),
        "cost_per_task":          round(est.cost_usd / tasks, 4),
    }

def print_report(hours: float):
    print(f"\n{'='*55}")
    print(f"  Claude Code Session Cost Report — {datetime.now():%Y-%m-%d %H:%M}")
    print(f"{'='*55}")

    for model in MODEL_PRICING:
        est = estimate_session(hours, model)
        tier = "★★★" if "opus" in model else "★★ " if "sonnet" in model else "★  "
        print(f"\n{tier} {model}")
        print(f"  Tasks:        ~{est['estimated_tasks']}")
        print(f"  Tokens:       {est['total_tokens']:,} ({est['input_tokens']:,} in / {est['output_tokens']:,} out)")
        print(f"  Session cost: ${est['estimated_cost_usd']:.4f}")
        print(f"  Cost/hour:    ${est['cost_per_hour']:.4f}")
        print(f"  Cost/task:    ${est['cost_per_task']:.4f}")

    print(f"\n{'─'*55}")
    print("  Developer hourly rate reference:")
    dev_hourly = 75  # USD, adjust to your rate
    print(f"  Dev rate ${dev_hourly}/hr × {hours}hrs = ${dev_hourly * hours:.0f}")
    sonnet_cost = estimate_session(hours, "claude-sonnet-4-20250514")["estimated_cost_usd"]
    print(f"  Sonnet cost ratio: {sonnet_cost / (dev_hourly * hours) * 100:.1f}% of dev time cost")
    print(f"{'='*55}\n")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--hours", type=float, default=5.0)
    args = parser.parse_args()
    print_report(args.hours)
```

```bash
# Run the report
python scripts/session-cost-report.py --hours 5
```

---

## 2. Model tier selection guide

```
Task                           Recommended model      Reason
─────────────────────────────────────────────────────────────────
Classification / labelling     Haiku                  Simple, fast, cheap
Extraction (JSON from text)    Haiku                  Deterministic task
Boilerplate generation         Haiku                  Template-driven
Simple refactors               Haiku / Sonnet          Low complexity
Code review                    Sonnet                 Needs judgment
Debugging complex issues       Sonnet                 Multi-step reasoning
Architecture design            Sonnet / Opus           Requires depth
Novel algorithm design         Opus                   Maximum reasoning
Complex security audit         Opus                   Can't miss anything
```

---

## 3. Context window management

Long contexts are expensive. Keep them lean:

```python
# scripts/estimate-context-size.py
# Run before a long Claude Code session to estimate token usage

from pathlib import Path
import sys

def estimate_tokens(text: str) -> int:
    """Rough estimate: 1 token ≈ 4 characters for English/code."""
    return len(text) // 4

def audit_context(paths: list[str]) -> None:
    total = 0
    print("Context size audit:")
    for path_str in paths:
        path = Path(path_str)
        if path.is_file():
            size = estimate_tokens(path.read_text(errors="replace"))
            total += size
            flag = " ⚠️ LARGE" if size > 2000 else ""
            print(f"  {size:6,} tokens  {path}{flag}")
        elif path.is_dir():
            for f in path.rglob("*.ts"):
                if "node_modules" not in str(f):
                    size = estimate_tokens(f.read_text(errors="replace"))
                    total += size

    print(f"\n  Total:  {total:,} tokens")
    print(f"  Cost (Sonnet input): ${total * 3 / 1_000_000:.4f}")

    if total > 50_000:
        print("\n  ⚠️  Context is large. Consider:")
        print("     - Summarise long files before including")
        print("     - Use only the relevant sections")
        print("     - Switch to Haiku for this session")

if __name__ == "__main__":
    audit_context(sys.argv[1:] or ["."])
```

```bash
# Check context size before a big session
python scripts/estimate-context-size.py src/
```

---

## 4. Prompt compression tips

```
BEFORE (expensive):
  "Here is the full contents of our 800-line UserService class.
   Please review it for bugs and suggest improvements.
   [800 lines of code]"

AFTER (cheaper — ~60% fewer tokens):
  "Review this function for bugs. Focus on: error handling, N+1 queries, auth.
   [only the 40-line function that matters]"

Rules:
  1. Include only the code Claude needs to see — not the whole file
  2. Use line ranges: "Look at lines 45-80 of src/users/service.ts"
  3. Summarise context: "We have a REST API with Express, Prisma, PostgreSQL"
     instead of pasting config files
  4. Ask for diffs, not rewrites: "Show me only the lines that need changing"
  5. Use structured prompts: what to do, what to ignore, expected output format
```

---

## 5. Monthly cost tracking template

```markdown
# AI Development Cost Tracker — [Month Year]

## Summary
| Week | Sessions | Hours | Model Mix | Cost USD |
|------|----------|-------|-----------|----------|
| W1   | 5        | 20h   | 80% Sonnet, 20% Haiku | $2.40 |
| W2   | 6        | 24h   | 70% Sonnet, 30% Haiku | $2.80 |
| W3   | 4        | 16h   | 90% Sonnet, 10% Haiku | $2.10 |
| W4   | 5        | 20h   | 75% Sonnet, 25% Haiku | $2.50 |
| **Total** | **20** | **80h** | | **$9.80** |

## ROI Calculation
- Developer hourly cost (fully loaded): $75/hr
- Hours saved by AI assistance (est): 20h × 30% productivity gain = 6h saved
- Value of time saved: 6h × $75 = $450
- AI cost: $9.80
- **ROI: 45:1**

## Anomalies
- W2 session on 2024-01-14: $1.20 for one session (unusually high)
  - Cause: Large codebase loaded for architecture review (context audit recommended)
  - Fix: Pre-summarise large files before including in context

## Optimisations this month
- Switched extraction tasks to Haiku: saved ~$0.80/week
- Started using line ranges instead of full files: ~20% token reduction
```

---

## 6. Team usage guidelines

```markdown
## Claude Code Usage Guidelines

### Model selection
- **Default:** claude-sonnet-4-20250514 for all coding tasks
- **Haiku:** classification, simple extraction, boilerplate, single-function tasks
- **Opus:** architecture decisions, security audits, novel problem-solving only

### Context hygiene
- Never paste full files if only a function is needed
- Use `# relevant section` comments to mark what Claude should focus on
- Clear context between unrelated tasks (start a new session)

### Session budgets
- Individual developer: no hard limit, but aim for < $5/day
- Alert threshold: > $2 for a single session (check for context bloat)
- Team monthly target: < $50/developer

### What NOT to use Claude Code for
- Generating large boilerplate that a template or scaffold command does better
- Tasks that take 30 seconds to Google (waste of tokens + slower)
- Iterating the same prompt 10+ times (rewrite the prompt instead)
```
