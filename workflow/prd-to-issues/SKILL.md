---
name: prd-to-issues
description: >
  Convert a PRD (Product Requirements Document) into a set of granular, actionable
  GitHub issues with labels, estimates, and dependency links. Use when the user has
  a PRD or spec and wants to break it into dev tasks.
  Triggers: "turn this PRD into issues", "break this into tasks", "create GitHub issues
  from this spec", "prd to issues", "decompose this feature".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# PRD to Issues Skill

## Process

### Step 1 — Read and understand the PRD
Read the full PRD carefully. If it's in a file, read it. If the user pastes it,
confirm understanding before proceeding.

```bash
# If PRD is in the repo
cat docs/prd/*.md 2>/dev/null || cat SPEC.md 2>/dev/null
```

### Step 2 — Identify issue types
Categorise work into:
- **feat**: New functionality
- **chore**: Setup, config, scaffolding
- **test**: Test coverage tasks
- **docs**: Documentation
- **refactor**: Preparatory refactoring

### Step 3 — Decompose into atomic issues
Each issue must be:
- Completable by one developer in ≤ 1 day
- Has a clear definition of done
- Contains no ambiguity about what "done" looks like
- Tests external behaviour, not implementation details

### Step 4 — Create issues with dependencies

```bash
# Create each issue and capture its number
ISSUE_1=$(gh issue create \
  --title "feat: [first task]" \
  --body "## What
[specific deliverable]

## Acceptance criteria
- [ ] [criterion 1]
- [ ] [criterion 2]

## Notes
[any implementation notes]" \
  --label "feat" \
  --assignee "@me" | grep -oP '#\K[0-9]+')

# Create dependent issue
gh issue create \
  --title "feat: [second task]" \
  --body "## What
[specific deliverable]

## Depends on
- #${ISSUE_1}

## Acceptance criteria
- [ ] [criterion 1]" \
  --label "feat"
```

### Step 5 — Add to project board

```bash
# Add all created issues to the project
gh project item-add [PROJECT_NUMBER] --owner @me --url $(gh issue list --json url -q '.[].url' | head -10)
```

### Issue template
```markdown
## What
[One paragraph describing exactly what this issue delivers]

## Why
[Why this is needed — link to PRD or parent issue]

## Acceptance criteria
- [ ] [Specific, testable criterion]
- [ ] [Another criterion]
- [ ] Tests cover: [what scenarios]

## Out of scope
[What this issue does NOT cover]

## Depends on
- #[issue number] (if applicable)
```
