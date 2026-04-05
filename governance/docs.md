---
name: governance-docs
description: Technical documentation generation — Architecture Decision Records (ADRs), API documentation, runbooks, and onboarding guides. Use to auto-generate docs from code/PRs, create ADR templates, produce OpenAPI docs, or write team onboarding materials.
---

The user needs to generate or structure technical documentation. Apply the relevant section.

## Architecture Decision Records (ADRs)

### ADR format (MADR — Markdown Architectural Decision Records)
```markdown
# ADR-{number}: {Short title of the decision}

**Status**: Proposed | Accepted | Deprecated | Superseded by ADR-{n}
**Date**: {YYYY-MM-DD}
**Deciders**: {names or team}
**Context tags**: `{technology}` `{domain}` `{concern}`

## Context and Problem Statement

{Describe the context and problem in 2–4 sentences. What forces are at play?
What is the architectural question we are addressing?}

## Decision Drivers

- {driver 1 — e.g. "Must support 10K concurrent users"}
- {driver 2 — e.g. "Team has no Go experience"}
- {driver 3 — e.g. "Must integrate with existing PostgreSQL"}

## Considered Options

- Option A: {name}
- Option B: {name}
- Option C: {name}

## Decision Outcome

**Chosen option**: {Option X}, because {1–2 sentences of primary justification}.

### Positive Consequences

- {e.g. "Reduces infrastructure cost by ~40%"}
- {e.g. "Aligns with team's existing TypeScript expertise"}

### Negative Consequences / Risks

- {e.g. "Vendor lock-in to AWS — migration cost if we switch cloud"}
- {e.g. "Requires learning curve for tRPC — ~2 weeks ramp-up"}

## Pros and Cons of the Options

### Option A: {name}
- ✅ {advantage}
- ✅ {advantage}
- ❌ {disadvantage}

### Option B: {name}
- ✅ {advantage}
- ❌ {disadvantage}
- ❌ {disadvantage}

## Links

- {Link to related ADR, RFC, or design doc}
- {Link to the PR or spike that informed this decision}
```

### ADR from PR context (prompt template for Claude)
```
Given the following PR description and diff, generate an ADR that captures
the architectural decision made.

PR Title: {title}
PR Description: {description}
Key files changed: {file list}
Diff summary: {diff}

Generate a complete MADR-format ADR. Infer the decision drivers from the
PR context. List realistic alternatives that were likely considered.
```

### ADR file naming and storage
```
docs/
  decisions/
    adr-001-use-postgresql-over-mysql.md
    adr-002-adopt-trpc-for-internal-apis.md
    adr-003-migrate-to-eks-from-ecs.md
    README.md    ← index of all ADRs with one-line summary
```

## API Documentation

### OpenAPI from code (automatic)
```ts
// Fastify + @fastify/swagger → auto-generates OpenAPI 3.1 spec
await fastify.register(fastifySwagger, {
  openapi: {
    info: { title: 'Orders API', version: '1.0.0' },
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' }
      }
    }
  }
})
await fastify.register(fastifySwaggerUI, { routePrefix: '/docs' })

// Every route schema auto-populates the spec
fastify.get('/orders/:id', {
  schema: {
    tags: ['Orders'],
    summary: 'Fetch an order by ID',
    params: z.object({ id: z.string().uuid() }),
    response: {
      200: OrderSchema,
      404: ErrorSchema,
    },
    security: [{ bearerAuth: [] }],
  },
  handler: getOrderHandler,
})
```

### API documentation structure (for public/partner APIs)
```markdown
# {Service Name} API Reference

## Authentication
{How to obtain and use tokens. Code example for each SDK.}

## Base URL
Production: `https://api.company.com/v1`
Staging:    `https://api-staging.company.com/v1`

## Rate Limits
| Tier | Requests/minute | Burst |
|---|---|---|
| Standard | 100 | 200 |
| Premium | 1000 | 2000 |

Headers returned: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Error Codes
| Code | HTTP Status | Meaning |
|---|---|---|
| `VALIDATION_FAILED` | 400 | Request body failed validation |
| `UNAUTHENTICATED` | 401 | Missing or invalid token |

## Endpoints
{One section per resource group — generated from OpenAPI spec}
```

## Runbook Template

```markdown
# Runbook: {Service Name} — {Operation or Incident Type}

**Owner**: {Team}
**Last reviewed**: {YYYY-MM-DD}
**Applies to**: {service-name} v{version}+

## Purpose
{1–2 sentences. What does this runbook help the on-call engineer do?}

## Prerequisites
- [ ] Access to {tool/console/dashboard}
- [ ] Membership in {PagerDuty rotation / Slack channel}
- [ ] VPN connected (if required)

## Step-by-Step Procedure

### Step 1: Verify the problem
```bash
# Check error rate
kubectl logs -n production deployment/{service} --tail=100 | grep ERROR | wc -l

# Check pod status
kubectl get pods -n production -l app={service}
```

### Step 2: {action}
{Instructions. Include exact commands with placeholders for variables.}

### Step 3: Validate resolution
```bash
# Expected output after fix
curl -sf https://api.company.com/health | jq .
# {"status": "ok", "version": "1.2.3"}
```

## Escalation
If unresolved after 30 minutes: page {team} via PagerDuty
If data loss suspected: immediately notify {data-owner} and pause writes

## Related Resources
- Dashboard: {Grafana URL}
- Logs: {CloudWatch / Loki URL}
- Related ADR: ADR-{n}
- Post-mortem template: docs/incidents/postmortem-template.md
```

## Onboarding Guide Generator

```
Prompt template for generating team onboarding docs from codebase:

"Analyse the following repository structure, README, and key files.
Generate a comprehensive onboarding guide for a new backend engineer.
Include:
1. System overview (what does this service do and why does it exist?)
2. Local development setup (step-by-step, copy-pasteable commands)
3. Architecture overview (key components and how they interact)
4. Key workflows (how does a typical feature get built and deployed?)
5. Where to find things (file structure map with annotations)
6. Common pitfalls (top 5 mistakes new engineers make)
7. Who to ask (team contacts by domain)

Repository: {contents}"
```

## Output Format

For ADR generation:
1. Fill in every section of the MADR template — no placeholders left empty
2. Infer decision drivers from the context provided
3. Always include at least 2 alternatives with honest pros/cons

For API docs:
1. Show the schema definition first, then the prose description
2. Include curl + SDK examples for every endpoint

For runbooks:
1. Use exact commands, not conceptual steps
2. Include the expected output of each command so the engineer can verify
