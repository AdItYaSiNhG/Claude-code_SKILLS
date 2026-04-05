---
name: integrations-github
description: GitHub automation — Conventional Commits enforcement, semantic versioning, CHANGELOG generation, release notes, and PR workflow automation. Use to implement a complete release engineering workflow from commit message to published release.
---

The user is automating GitHub workflows. This skill covers the full commit-to-release pipeline.

## Conventional Commits

### Specification
```
<type>(<scope>): <short description>

[optional body]

[optional footer(s)]
```

**Types** (use exactly these):
| Type | Triggers | Use for |
|---|---|---|
| `feat` | minor version bump | New feature |
| `fix` | patch version bump | Bug fix |
| `feat!` or `BREAKING CHANGE:` footer | major version bump | Breaking change |
| `chore` | no bump | Tooling, config, deps |
| `docs` | no bump | Documentation only |
| `style` | no bump | Formatting, no logic change |
| `refactor` | no bump | Restructure, no behaviour change |
| `perf` | patch | Performance improvement |
| `test` | no bump | Adding/fixing tests |
| `ci` | no bump | CI/CD changes |
| `build` | no bump | Build system changes |

**Examples**:
```
feat(orders): add bulk order cancellation endpoint

Allows cancelling up to 50 orders in a single API call.
Implements idempotency via client-supplied request ID.

Closes #234

---

fix(auth): prevent token refresh race condition

Multiple concurrent requests could each attempt a token refresh,
causing some to fail with 401. Added a mutex to serialise refresh calls.

---

feat!: remove deprecated v1 order endpoints

BREAKING CHANGE: /v1/orders and /v1/orders/:id have been removed.
Migrate to /v2/orders — see migration guide in docs/v2-migration.md.
```

## Commit Linting (CI enforcement)

```bash
# Install
npm install -D @commitlint/cli @commitlint/config-conventional

# commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'scope-enum': [2, 'always', [
      'orders', 'auth', 'payments', 'users', 'api', 'db', 'ci', 'deps'
    ]],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [1, 'always', 120],
  },
}
```

```yaml
# .github/workflows/commitlint.yml
name: Commitlint
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: wagoid/commitlint-github-action@v5
```

### Commitizen (interactive commit helper)
```bash
npm install -D commitizen cz-conventional-changelog
# package.json
"scripts": { "commit": "cz" },
"config": { "commitizen": { "path": "cz-conventional-changelog" } }

# Developers run: npm run commit  (instead of git commit)
```

## Semantic Versioning & CHANGELOG

### release-please (Google — recommended)
```yaml
# .github/workflows/release-please.yml
name: Release Please
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: node
          token: ${{ secrets.GITHUB_TOKEN }}
```

How it works:
1. Accumulates commits on `main` since last release
2. Opens a "Release PR" with bumped version + generated CHANGELOG
3. On PR merge: creates GitHub Release + git tag
4. CHANGELOG format follows Keep a Changelog standard

```json
// release-please-config.json (monorepo)
{
  "packages": {
    "packages/api": { "release-type": "node", "package-name": "@company/api" },
    "packages/ui":  { "release-type": "node", "package-name": "@company/ui" }
  }
}
```

### Changesets (alternative, better for monorepos with independent versioning)
```bash
# Developers run this when their PR includes a user-facing change
npx changeset

# Prompts:
# → Which packages are affected? (checkbox)
# → What kind of change? (major/minor/patch)
# → Describe the change: (release note text)

# Creates: .changeset/bright-dogs-run.md

# On main: CI consumes changesets, bumps versions, updates CHANGELOG
npx changeset version  # bump
npx changeset publish  # publish to npm
```

```yaml
# .github/workflows/release.yml
- name: Create Release PR or Publish
  uses: changesets/action@v1
  with:
    publish: npm run build && changeset publish
    version: changeset version
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## CHANGELOG Generation

### Keep a Changelog format
```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [2.1.0] - 2025-04-01

### Added
- Bulk order cancellation endpoint (up to 50 orders per request)
- Customer segment field in order analytics export

### Changed
- Order search now returns results sorted by relevance by default

### Fixed
- Token refresh race condition causing intermittent 401 errors

### Security
- Updated jsonwebtoken to patch CVE-2024-XXXX

## [2.0.0] - 2025-02-15

### BREAKING CHANGES
- Removed deprecated v1 order endpoints — see migration guide
- `order.customerId` field renamed to `order.customer_id`
```

### Auto-generate CHANGELOG from commits (for projects not using release-please)
```bash
# conventional-changelog-cli
npx conventional-changelog-cli -p angular -i CHANGELOG.md -s -r 0

# Or: git-cliff (Rust, faster, highly configurable)
# cliff.toml
[changelog]
header = "# Changelog\n"
body = """
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | striptags | trim | upper_first }}
{% for commit in commits %}- {{ commit.message | upper_first }}{% endfor %}
{% endfor %}
"""
trim = true

[git]
conventional_commits = true
commit_parsers = [
  { message = "^feat", group = "Added" },
  { message = "^fix", group = "Fixed" },
  { message = "^perf", group = "Performance" },
  { message = "^refactor", group = "Changed" },
  { message = "^.*!|BREAKING", group = "Breaking Changes", default_scope = "all" },
]
```

## PR Automation

### Auto-assign reviewers by CODEOWNERS
```
# .github/CODEOWNERS
/src/api/         @company/backend-team
/src/ui/          @company/frontend-team
/infra/           @company/platform-team
*.sql             @company/dba-team
/docs/decisions/  @company/architects
```

### PR template
```markdown
<!-- .github/pull_request_template.md -->
## Summary
<!-- What does this PR do? -->

## Type of change
- [ ] Bug fix (non-breaking)
- [ ] New feature (non-breaking)
- [ ] Breaking change
- [ ] Documentation
- [ ] Chore / refactor

## Checklist
- [ ] Tests added / updated
- [ ] CHANGELOG entry added (or changeset created)
- [ ] ADR created if architectural decision made
- [ ] No PII or secrets in diff
- [ ] Dependent PRs linked

## Related issues
Closes #
```

### Release notes generation (LLM-powered via Claude Code)
```bash
# Script: scripts/generate-release-notes.sh
LAST_TAG=$(git describe --tags --abbrev=0)
COMMITS=$(git log $LAST_TAG..HEAD --pretty=format:"%s" --no-merges)

claude -p "
You are a technical writer. Given these conventional commits since the last release,
generate user-facing release notes for our engineering blog.
Group by feature area, not commit type.
Write for a technical audience — be specific, not vague.
Include the most impactful changes prominently.

Commits:
$COMMITS

Output format: Markdown with ## headers for each section.
"
```

## Output Format

1. For commit conventions: show the exact commitlint config + examples for their scopes
2. For release pipeline: show the full release-please or changesets workflow YAML
3. For CHANGELOG: generate the actual CHANGELOG entry from commits provided
4. For PR automation: show CODEOWNERS + PR template ready to commit
