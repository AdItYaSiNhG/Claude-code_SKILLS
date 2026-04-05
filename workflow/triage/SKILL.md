---
name: triage-issue
description: >
  Investigate a bug by exploring the codebase, identify the root cause, and file
  a GitHub issue with a TDD-based fix plan. Use when the user reports a bug and
  wants a proper investigation before fixing.
  Triggers: "triage this bug", "investigate this issue", "root cause", "file a bug",
  "create a bug report", "investigate why", "what's causing this".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Triage Issue Skill

## Process

### Step 1 — Gather information
Ask the user:
- What is the expected behaviour?
- What is the actual behaviour?
- Steps to reproduce
- Error messages or stack traces (exact text)
- When did this start? (recent change? always?)

### Step 2 — Reproduce the bug

```bash
# Run the failing test or reproduce scenario
npm test 2>/dev/null | grep -A10 "FAIL\|Error\|failed"

# Check recent git changes that could have introduced it
git log --oneline -20
git diff HEAD~5 -- src/affected-module.ts 2>/dev/null | head -60
```

### Step 3 — Locate the root cause

```bash
# Trace the code path
grep -rn "relevant_function\|error_message" src/ | grep -v test | head -20

# Read the failing module
cat src/affected-module.ts

# Check for recent dependency changes
git log --oneline -- package-lock.json package.json | head -10
git diff HEAD~3 -- package.json 2>/dev/null
```

### Step 4 — Write a failing test (TDD approach)
Before fixing, write a test that reproduces the bug:

```typescript
// This test should FAIL on the current code
it("reproduces bug: [description]", () => {
  // exact reproduction of the failing scenario
  expect(buggyFunction(input)).toBe(expectedOutput);
});
```

```bash
# Confirm the test fails
npx jest --testNamePattern="reproduces bug" 2>/dev/null | tail -20
```

### Step 5 — File the GitHub issue

```bash
gh issue create \
  --title "bug: [concise description]" \
  --label "bug" \
  --body "$(cat << 'EOF'
## Bug description
[What is wrong]

## Steps to reproduce
1. [Step 1]
2. [Step 2]
3. [See error]

## Expected behaviour
[What should happen]

## Actual behaviour
[What happens instead]

## Root cause
[What you found in the code — module, line, reason]

## Proposed fix
[High-level approach]

## Fix plan (TDD)
- [ ] Failing test added at: `tests/[path].test.ts`
- [ ] Fix implementation in: `src/[path].ts`
- [ ] Regression test added
- [ ] All existing tests still pass

## Environment
- Node/Python/Go version:
- OS:
- Commit where it broke: `git bisect` result if known
EOF
)"
```

### Step 6 — Optional: implement the fix now
If the user wants the fix immediately:
1. Keep the failing test from Step 4
2. Implement the minimum fix to make it pass
3. Verify no regressions: `npm test`
4. Commit: `git commit -m "fix: [description] (closes #ISSUE_NUMBER)"`
