---
name: request-refactor-plan
description: >
  Explore the codebase and produce a safe, incremental refactoring plan with
  prioritised steps, risk assessment, and rollback strategy. Use when the user
  wants to refactor something but isn't sure how to break it down safely.
  Triggers: "refactor plan", "how to refactor", "plan a refactor", "safe refactor",
  "improve this code", "break down refactor", "refactoring strategy".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Request Refactor Plan Skill

## Process

### Step 1 — Understand the problem
Ask: what is wrong with the current code? What outcome do we want?
Read the file(s) in question before making any suggestions.

```bash
# Read the target
cat src/the-file-to-refactor.ts

# Check test coverage
npx jest --coverage --collectCoverageFrom="src/the-file*" 2>/dev/null | tail -20
```

### Step 2 — Assess the current state

```bash
# Complexity
radon cc . -n C 2>/dev/null | head -20              # Python
npx eslint --rule '{"complexity":["warn",5]}' 2>/dev/null # JS/TS

# Test coverage — DO NOT refactor untested code without adding tests first
npx jest --coverage 2>/dev/null | grep -A2 "the-file"

# Dependents — who will break if the interface changes?
npx madge --dependents src/the-file.ts 2>/dev/null
grep -rn "import.*the-file\|from.*the-file" src/ | grep -v test | wc -l
```

### Step 3 — Produce a phased plan

Output a plan in this format:

```markdown
## Refactor Plan: [Name]

**Goal:** [what will be better]
**Risk:** Low / Medium / High
**Estimated time:** X hours / Y days

### Phase 1 — Safety net (no behaviour change)
Prerequisites before any refactoring:
- [ ] Add tests to reach 80% coverage on affected code
- [ ] Fix existing type errors in scope
- [ ] Ensure CI is green on current main

### Phase 2 — Preparatory moves (behaviour-preserving)
Small changes that make the real refactor easier:
- [ ] Extract [X] into separate function
- [ ] Rename [Y] for clarity
- [ ] Move [Z] to correct module

Commit after each step. Each step must keep tests green.

### Phase 3 — Core structural change
The actual refactor:
- [ ] [Specific change]
- [ ] [Another change]

### Phase 4 — Cleanup
- [ ] Remove dead code
- [ ] Update imports
- [ ] Update documentation

### Rollback plan
Each phase is a separate PR. Phase N can be reverted independently.

### Verification
After every commit: `npm test && npm run build`
After every PR: smoke test on staging
```

### Step 4 — Start with Phase 1 only
Never start coding Phase 2+ until Phase 1 is complete and merged.

```bash
# Always verify tests pass before and after each step
npm test 2>/dev/null | tail -10
```
