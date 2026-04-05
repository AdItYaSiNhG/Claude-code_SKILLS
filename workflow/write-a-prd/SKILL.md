---
name: write-a-prd
description: >
  Create a PRD through user interview, codebase exploration, and module design,
  then submit as a GitHub issue. Use when the user wants to write a PRD, create
  a product requirements document, or plan a new feature.
  Triggers: "write a PRD", "product requirements", "plan this feature", "PRD for",
  "I want to build", "create a spec", "feature spec".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Write a PRD Skill

## Process

This skill is invoked when the user wants to create a PRD. You may skip steps
that are not necessary for the scope of the request.

### Step 1 — Deep user interview
Ask the user for a long, detailed description of:
- The problem they want to solve
- Who experiences the problem (users, personas)
- Any ideas they already have for solutions
- What success looks like

Do NOT proceed until you have a thorough understanding of the problem space.

### Step 2 — Codebase exploration
Before proposing solutions, explore the repo:
```bash
# Understand what already exists
find . -type f -name "*.ts" -o -name "*.py" | grep -v node_modules | head -30
# Verify the user's assertions about the current state
grep -rn "relevant_feature" src/ 2>/dev/null | head -20
```
Verify the user's assertions. Understand what modules already exist and what
would need to be built or modified.

### Step 3 — Relentless design interview
Interview the user about every aspect of the design until you reach a shared
understanding. Walk down each branch of the design tree, resolving dependencies
between decisions one-by-one. Ask:
- What happens when X fails?
- What does the user see when Y is loading?
- How does this interact with Z (existing feature)?
- What are the edge cases?

### Step 4 — Module design
Sketch out the major modules you will need to build or modify:
- Identify opportunities to extract deep, well-tested modules
- Look for concerns that can be separated and tested in isolation
- Flag dependencies between modules
- Note which modules already exist vs. must be created

### Step 5 — Write the PRD
Once you have a complete shared understanding, write the PRD using this template
and submit as a GitHub issue:

```markdown
## Problem
[The problem that the user is facing, from the user's perspective]

## Solution
[The solution to the problem, from the user's perspective]

## User Stories
[A LONG, numbered list. Each story: "As a <persona>, I want <capability>, so that <benefit>"]
1. As a [persona], I want [feature], so that [outcome]
2. ...
(minimum 10 stories, cover all aspects of the feature)

## Implementation Decisions
[Decisions made during design. Do NOT include specific file paths or code snippets — they go stale.]
- Decision: [what was decided]
  Rationale: [why]
- ...

## Testing Decisions
- What makes a good test for this feature (test external behaviour, not implementation)
- What scenarios must be tested
- What should NOT be tested (implementation details)

## Out of Scope
[What is explicitly not included in this PRD]
```

```bash
# Submit as GitHub issue
gh issue create \
  --title "feat: [feature name]" \
  --body-file /tmp/prd.md \
  --label "PRD,feature"
```
