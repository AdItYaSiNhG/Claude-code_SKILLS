---
name: devops
description: Enterprise DevOps skills covering CI/CD pipelines, multi-cloud infrastructure, and production observability/SRE. Use for pipeline design, IaC, cloud cost optimisation, SLO definition, and incident management.
license: Internal enterprise use
---

Three sub-skills cover the DevOps surface area:

| Sub-skill file | Invoke when… |
|---|---|
| `ci-cd.md` | designing or troubleshooting build, test, and deployment pipelines |
| `cloud.md` | provisioning infrastructure, managing multi-cloud resources, or FinOps |
| `observability.md` | setting up monitoring, defining SLOs, or managing incidents |

## Routing

- Pipeline is broken or slow → `ci-cd.md`
- Infrastructure provisioning or cost → `cloud.md`
- Something is wrong in production → `observability.md`
- Full platform build → all three in sequence

## Enterprise Context

DevOps failures at scale cluster around: flaky tests blocking deploys, unreviewed infrastructure drift, missing SLOs making incidents ambiguous, and runaway cloud spend. These skills address each systematically.
