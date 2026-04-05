---
name: governance
description: Enterprise AI governance skills covering Claude Code token/cost management, usage compliance, and technical documentation generation. Use for controlling Claude Code spend, enforcing usage policies, generating ADRs, and automating documentation from code.
license: Internal enterprise use
---

Four sub-skills cover governance concerns:

| Sub-skill file | Invoke when… |
|---|---|
| `tokens.md` | analysing or optimising token usage and session costs in Claude Code |
| `cost.md` | building cost dashboards, budget alerts, or chargeback reports |
| `compliance.md` | enforcing acceptable-use policies, data handling rules, or audit trails |
| `docs.md` | generating ADRs, API docs, runbooks, or onboarding documentation |

## Governance Mandate

Governance is what makes Claude Code sustainable at enterprise scale. Without it: costs balloon uncontrollably, sensitive data leaks into prompts, and institutional knowledge stays trapped in engineers' heads. These skills systematise all three.

## Routing

- "How much did we spend?" → `cost.md`
- "Why is my session so long/expensive?" → `tokens.md`
- "Is this usage acceptable?" → `compliance.md`
- "Document this code / create an ADR" → `docs.md`
