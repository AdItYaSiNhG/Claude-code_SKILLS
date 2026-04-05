---
name: devops-ci-cd
description: CI/CD pipeline design, optimisation, and troubleshooting across GitHub Actions, GitLab CI, and Jenkins. Covers pipeline architecture, caching strategies, test parallelisation, deployment strategies (blue-green, canary, GitOps), and pipeline security.
---

The user is designing, optimising, or debugging a CI/CD pipeline. Apply the relevant section below.

## Pipeline Architecture

### Recommended Stage Order
```
trigger → lint+typecheck → unit-tests → build → integration-tests → security-scan → deploy-staging → e2e-tests → deploy-production
```

Key rules:
- Fail fast: lint and typecheck run first, in parallel, before any build
- Never deploy to production from a branch — only from `main` / release tags
- Security scans (SAST, SCA) block production deploy if high/critical findings
- E2E tests run against staging, not production

### GitHub Actions — Production Template

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm run lint && pnpm run typecheck
      - run: pnpm run test:unit --coverage
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }

  build:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm run build
      - uses: actions/upload-artifact@v4
        with: { name: dist, path: dist/ }

  security:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: SAST (Semgrep)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/owasp-top-ten
      - name: SCA (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: HIGH,CRITICAL
          exit-code: 1

  deploy-staging:
    needs: [build, security]
    if: github.ref == 'refs/heads/main'
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist }
      - run: ./scripts/deploy.sh staging

  e2e:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm exec playwright install --with-deps
      - run: pnpm run test:e2e
        env:
          BASE_URL: ${{ vars.STAGING_URL }}

  deploy-production:
    needs: e2e
    environment: production         # requires manual approval in GitHub UI
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/deploy.sh production
```

## Caching Strategy

```yaml
# Node.js — cache node_modules keyed to lockfile hash
- uses: actions/cache@v4
  with:
    path: ~/.pnpm-store
    key: pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: pnpm-

# Docker layer caching — use GitHub Actions cache backend
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max

# Go modules
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: go-${{ hashFiles('go.sum') }}

# Python (uv)
- uses: actions/cache@v4
  with:
    path: ~/.cache/uv
    key: uv-${{ hashFiles('uv.lock') }}
```

**Target cache hit rate**: >85% on PRs. If below 70%, the key is too specific.

## Test Parallelisation

```yaml
# GitHub Actions matrix — split test suite
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: pnpm vitest --shard=${{ matrix.shard }}/4

# Playwright — parallel browsers
  e2e:
    strategy:
      matrix:
        project: [chromium, firefox, webkit]
    steps:
      - run: pnpm playwright test --project=${{ matrix.project }}
```

**For large test suites**: use `jest --testPathPattern` or `pytest -k` with dynamic sharding based on historical test duration (Buildkite Test Analytics or Gradle test retry).

## Deployment Strategies

### Blue-Green (zero downtime, instant rollback)
```bash
# 1. Deploy new version to green environment
deploy green $IMAGE_TAG

# 2. Run smoke tests against green
curl -f https://green.internal/health

# 3. Swap load balancer (< 1s switchover)
aws elbv2 modify-listener --listener-arn $LB_ARN \
  --default-actions Type=forward,TargetGroupArn=$GREEN_TG

# 4. Keep blue warm for 30 min, then decommission
```

### Canary (progressive rollout, metrics-gated)
```yaml
# Argo Rollouts canary strategy
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5      # 5% traffic
        - pause: { duration: 5m }
        - analysis:         # auto-rollback if error rate > 1%
            templates:
              - templateName: error-rate
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
```

### GitOps (ArgoCD / Flux)
```
Repo structure:
  apps/
    production/
      values.yaml          # overrides for prod
    staging/
      values.yaml
  base/
    deployment.yaml
    service.yaml

ArgoCD App:
  spec:
    source:
      repoURL: https://github.com/org/infra
      path: apps/production
    destination:
      server: https://k8s-prod.internal
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
```

## Pipeline Security

```yaml
# Principle of least privilege — per-job OIDC tokens, not long-lived secrets
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/github-actions-deploy
    aws-region: ap-south-1

# Pin ALL third-party actions to a commit SHA (not a tag)
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# Secret scanning — block secrets from entering the repo
- uses: trufflesecurity/trufflehog@v3
  with:
    only-verified: true
```

## Pipeline Performance Benchmarks

| Metric | Target | Alert threshold |
|---|---|---|
| PR feedback time (lint+unit) | < 3 min | > 5 min |
| Full pipeline (unit→deploy-staging) | < 15 min | > 20 min |
| Cache hit rate | > 85% | < 70% |
| Flaky test rate | < 0.5% | > 2% |
| Deploy frequency | Daily+ | < Weekly |

## Flaky Test Management

```bash
# Identify flaky tests (GitHub Actions)
gh run list --workflow ci.yml --limit 50 --json conclusion,jobs \
  | jq '[.[].jobs[] | select(.conclusion == "failure")] | group_by(.name) | map({name: .[0].name, failures: length})'

# Quarantine pattern — tag flaky tests, run separately
# pytest: use @pytest.mark.flaky(reruns=3)
# Jest: use jest-circus with retryTimes(3)
# Playwright: use retries: 2 in config
```

## Output Format

1. Show the full pipeline YAML for the user's CI platform (GitHub Actions / GitLab CI)
2. Highlight caching opportunities specific to their stack
3. Flag any security gaps (unpinned actions, long-lived secrets, missing approval gates)
4. Provide the deploy strategy recommendation based on their risk tolerance
