---
name: code-review
description: >
  Use this skill for all code review tasks: PR review checklists, diff-level feedback,
  architecture review, performance hotspot review, security-focused review, API contract
  review, database query review, concurrency & race condition review, review summary
  generation, and review SLA tracking.
  Triggers: "review this", "review my PR", "review this diff", "code review",
  "check my changes", "review this function", "architecture review", "API review".
compatibility: "Claude Code (claude.ai, Claude Desktop) — works with diffs, files, or full repos"
version: "1.0.0"
---

# Code Review Skill

## Why this skill exists

Ad-hoc review misses whole categories of issues. This skill applies a
**consistent, multi-lens review**: correctness → security → performance →
maintainability → API design → testing → documentation.

When reviewing a diff or PR, always produce:
1. A **one-paragraph summary** of what the change does
2. **Categorised findings** (grouped by severity)
3. **Inline suggestions** with code examples
4. A **final verdict**: Approve / Request Changes / Needs Discussion

---

## 0. Context gathering

Before reviewing, understand the change:

```bash
# If reviewing a PR from a local repo
git log --oneline -1
git diff main...HEAD --stat
git diff main...HEAD -- '*.ts' '*.py' '*.go' | head -300

# PR description / linked issue
gh pr view --json title,body,additions,deletions,changedFiles 2>/dev/null

# What tests exist for changed code?
git diff main...HEAD --name-only \
  | grep -v "test\|spec" \
  | while read f; do
    base=$(basename "$f" | sed 's/\.[^.]*$//')
    find . -name "*${base}*test*" -o -name "*${base}*spec*" 2>/dev/null | head -3
  done
```

---

## 1. Diff-level review checklist

Work through the diff systematically. For each changed file/function ask:

### Correctness
- [ ] Does the logic correctly implement the stated intent?
- [ ] Are edge cases handled? (null/undefined, empty collections, 0, negative numbers, max values)
- [ ] Are error paths handled? (network failures, DB errors, validation failures)
- [ ] Are return values / errors propagated correctly?
- [ ] Any off-by-one errors in loops or array access?
- [ ] Mutation of shared state done safely?

### Security
- [ ] No new hardcoded secrets / credentials
- [ ] All user input validated / sanitised before use
- [ ] No new SQL concatenation (use parameterised queries)
- [ ] Auth/authorisation not accidentally bypassed
- [ ] No new `eval()`, `exec()`, `dangerouslySetInnerHTML` without justification

### Performance
- [ ] No N+1 queries introduced (loop calling DB/API inside loop)
- [ ] No blocking I/O in async/event-loop code
- [ ] Large data sets paginated / streamed, not loaded into memory whole
- [ ] Expensive operations cached where appropriate
- [ ] Indexes exist for new DB queries

### Maintainability
- [ ] Functions are single-responsibility (≤20 lines ideal, ≤40 lines max)
- [ ] Cyclomatic complexity ≤ 10 per function
- [ ] No magic numbers / strings (use named constants)
- [ ] Duplication: does this logic already exist elsewhere?
- [ ] Naming is clear and consistent with the codebase

### Testing
- [ ] New logic has unit tests
- [ ] Tests cover happy path + at least 2 edge/error cases
- [ ] Mocks/stubs are isolated and not leaking between tests
- [ ] Test descriptions clearly state what's being tested

### Documentation
- [ ] Public API / exported functions have docstrings / JSDoc
- [ ] Non-obvious logic has inline comments explaining **why** not **what**
- [ ] README / API docs updated if behaviour changed

---

## 2. Quick diff scan commands

```bash
# Show diff with context
git diff main...HEAD -U5 2>/dev/null \
  || git diff HEAD~1 -U5

# Files changed
git diff main...HEAD --name-only 2>/dev/null | sort

# Lines added/removed
git diff main...HEAD --numstat 2>/dev/null

# Scan diff for common red flags
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "TODO|FIXME|HACK|XXX|console\.log|print\(|debugger|pdb\." \
  | head -20

# New hardcoded strings that look like secrets
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "(api_key|apikey|secret|password|token)\s*[:=]\s*['\"][^'\"]{8,}" \
  | head -10

# New SQL concatenation
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "query\s*\+|execute\s*\(.*\+|f['\"]SELECT|f['\"]INSERT|f['\"]UPDATE" \
  | head -10

# N+1 risk: DB call inside a loop
git diff main...HEAD 2>/dev/null \
  | grep -A5 "for.*in\|\.forEach\|\.map(" \
  | grep -E "await.*find|await.*query|\.findOne|db\." | head -10
```

---

## 3. Architecture review

For larger changes or design-level reviews:

```bash
# Module dependency graph — detect circular deps
npx madge --circular --extensions ts,js src/ 2>/dev/null

# Package coupling: what does this module import?
python3 - <<'EOF'
import re
from pathlib import Path
import sys

target = sys.argv[1] if len(sys.argv) > 1 else "src"
for f in Path(target).rglob("*.ts"):
    if "node_modules" in str(f): continue
    text = f.read_text(errors="replace")
    imports = re.findall(r"from ['\"]([^'\"]+)['\"]", text)
    external = [i for i in imports if not i.startswith(".")]
    if external:
        print(f"\n{f}:")
        for imp in external: print(f"  {imp}")
EOF
```

**Architecture review questions:**
- Does this change respect the existing layer boundaries? (controller → service → repository)
- Does it introduce new cross-layer dependencies?
- Is a new abstraction introduced where none was needed (over-engineering)?
- Is the change cohesive, or does it do too many things?
- Does it break the single-responsibility of any module?
- Are interfaces/contracts kept stable (backwards-compatible)?

---

## 4. Performance review

### Database query review
```bash
# Find new ORM calls in the diff
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "\.find|\.findOne|\.findMany|\.query|\.select|prisma\.\|knex\." \
  | head -20

# Check for missing pagination
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "\.find\(|\.findAll\(|\.select\(" \
  | grep -v "limit\|take\|skip\|offset\|paginate" | head -10

# Explain query (manual step — show the query and ask for EXPLAIN ANALYZE)
```

**Database review checklist:**
- [ ] New queries use indexes (check `EXPLAIN ANALYZE` for Seq Scan on large tables)
- [ ] SELECT specifies columns, not `SELECT *` on wide tables
- [ ] Bulk operations use batch inserts, not single-row loops
- [ ] Transactions wrap multi-step mutations
- [ ] Connection not left open across async awaits without releasing

### Frontend performance
```bash
# Bundle size impact — run after build
npx bundlesize 2>/dev/null
npx size-limit 2>/dev/null

# New lazy-load opportunities
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "import .* from" \
  | grep -v "type\|interface" \
  | head -20
```

---

## 5. Concurrency & race condition review

Key patterns to flag:

```bash
# Node.js — Promise.all vs sequential await in loops
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "for.*await|await.*forEach" | head -10

# Missing mutex / lock for shared state
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "shared|global|module\." \
  | grep -v "const\|readonly\|Object\.freeze" | head -10

# Python threading issues
git diff main...HEAD 2>/dev/null | grep "^+" \
  | grep -E "threading\.|multiprocessing\.|asyncio\." | head -10
```

**Concurrency red flags:**
- Reading and writing a shared variable without a lock
- Async function that does `const x = shared.val` then `await something()` then uses `x` — value may have changed
- `Promise.all` on operations that must be sequential (e.g., balance debit then credit)
- Non-atomic check-then-act patterns (TOCTOU)

---

## 6. API contract review

```bash
# Check for breaking changes in exported types / function signatures
git diff main...HEAD -- "*.ts" 2>/dev/null \
  | grep "^[-+]" \
  | grep -E "export (function|class|interface|type|const|enum)" \
  | head -30

# OpenAPI diff (if spec exists)
git diff main...HEAD -- "openapi*.yaml" "swagger*.yaml" "*.openapi.json" 2>/dev/null \
  | head -60
```

**API review checklist:**
- [ ] New endpoints follow existing naming conventions (RESTful, casing)
- [ ] Response shapes are consistent with other endpoints
- [ ] Breaking changes (removed fields, changed types) are versioned
- [ ] Error responses follow the standard error schema
- [ ] Pagination params are consistent (cursor vs offset)
- [ ] Rate limiting / auth applied to new endpoints

---

## 7. Review output template

Always produce review output in this format:

```markdown
## Code Review — [PR title / description]

**Summary:** [1–2 sentences describing what this change does and why]

**Files changed:** N  |  **Lines added:** +N  |  **Lines removed:** -N

---

### 🔴 Must Fix (blocking)

**[SECURITY] SQL injection risk**
`src/users/repo.ts:42`
```typescript
// current — dangerous
const result = await db.query(`SELECT * FROM users WHERE id = ${userId}`);

// suggested — safe
const result = await db.query("SELECT * FROM users WHERE id = $1", [userId]);
```
Reason: userId comes from user input and is not validated before use.

---

### 🟡 Should Fix (non-blocking but important)

**[PERFORMANCE] N+1 query in user list endpoint**
`src/users/service.ts:87`
The loop at line 87 calls `getPermissions(userId)` for each user.
With 1000 users this makes 1001 DB calls. Batch with `getPermissionsByUserIds(ids)`.

---

### 🔵 Nice to Have (minor suggestions)

**[STYLE] Magic number**
`src/auth/session.ts:15`
`3600` should be a named constant `SESSION_TTL_SECONDS = 3600`.

---

### ✅ Looks Good

- Error handling is thorough — all await calls wrapped in try/catch
- Tests cover happy path, invalid input, and not-found cases
- Naming is consistent with the rest of the module

---

### Verdict: 🟡 Request Changes

2 blocking issues to fix before merge. Happy to re-review.
```

---

## 8. Review severity guide

| Label | Criteria | Blocking? |
|-------|---------|----------|
| 🔴 Must Fix | Security bug, data loss risk, incorrect logic, broken API contract | Yes |
| 🟡 Should Fix | Performance issue, missing error handling, missing tests for non-trivial logic | Recommended |
| 🔵 Nice to Have | Style, naming, docs, minor refactor | No |
| ✅ Praise | Acknowledge good patterns — promotes them | — |
| ❓ Question | Need clarification before giving an opinion | Blocking pending answer |

---

## 9. Review SLA tracking

For teams tracking review velocity:

```bash
# PRs open >24h without review (requires gh CLI)
gh pr list --json number,title,createdAt,reviewDecision \
  --jq '.[] | select(.reviewDecision == "") |
    {number, title, age: ((now - (.createdAt | fromdateiso8601)) / 3600 | floor)}' \
  2>/dev/null | python3 -c "
import json, sys
for line in sys.stdin:
    try:
        pr = json.loads(line)
        age = pr.get('age', 0)
        flag = '🔴' if age > 48 else '🟡' if age > 24 else '🟢'
        print(f\"{flag} #{pr['number']} ({age}h) {pr['title'][:60]}\")
    except: pass
"
```
