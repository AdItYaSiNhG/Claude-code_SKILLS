---
name: backend
description: Enterprise backend engineering skills covering API design, database engineering, and language-specific runtime patterns. Use for server-side implementation, data layer decisions, or polyglot backend work.
license: Internal enterprise use
---

Three sub-skills cover the backend surface area:

| Sub-skill file | Invoke when… |
|---|---|
| `api.md` | designing or reviewing REST, GraphQL, gRPC, tRPC, or WebSocket APIs |
| `database.md` | schema design, query optimisation, migrations, or data pipeline work |
| `runtimes.md` | language-specific patterns for Node.js, Python, Go, Java, Rust, Ruby, or PHP |

## Routing Between Sub-skills

- API shape question → `api.md` first
- Performance or data modelling question → `database.md` first
- Language idiom, runtime, or toolchain question → `runtimes.md`
- Full-stack feature → `api.md` + `database.md` together

## Enterprise Context

Backend systems at scale fail on: unclear API contracts, migration risk on large tables, N+1 query patterns, and auth/authz gaps. These skills address all four systematically.
