---
name: code-cleanup
description: >
  Use this skill for code cleanup and refactoring tasks: legacy code modernisation,
  tech debt quantification, incremental refactor planning, dead dependency removal,
  unused env var cleanup, log noise reduction, error handling standardisation,
  i18n string extraction, and feature flag removal.
  Triggers: "clean up", "refactor", "tech debt", "legacy code", "dead code",
  "remove unused", "modernise", "i18n", "feature flag cleanup", "log cleanup".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Code Cleanup & Refactoring Skill

## Why this skill exists

Cleanup done without a plan causes more harm than good — broken imports,
missing test coverage, big-bang rewrites that stall. This skill provides
**safe, incremental, tool-backed cleanup** with verification steps at
each stage.

Golden rule: **every cleanup step must leave tests green**.

---

## 0. Pre-cleanup baseline

Before touching anything, capture the baseline:

```bash
# Record current test state
npx jest --coverage --json --outputFile=/tmp/baseline-tests.json 2>/dev/null \
  || pytest --cov=. --cov-report=json -q 2>/dev/null

# Record bundle/build size
npm run build 2>/dev/null && du -sh dist/ build/ 2>/dev/null
# or
wc -l $(find . -name "*.ts" -o -name "*.py" | grep -v node_modules) 2>/dev/null | tail -1

# Git checkpoint
git stash list | head -5
echo "Current branch: $(git branch --show-current)"
echo "Last commit: $(git log --oneline -1)"
```

Never start cleanup without a clean git working tree:
```bash
git status --short | head -10
# If dirty: git stash or commit first
```

---

## 1. Tech debt quantification

Convert debt into numbers leadership can act on:

```bash
# Count TODO/FIXME/HACK/DEBT markers
python3 - <<'EOF'
from pathlib import Path
import re
from collections import defaultdict

MARKERS = ["TODO", "FIXME", "HACK", "XXX", "DEBT", "DEPRECATED", "WORKAROUND"]
found = defaultdict(list)

for f in Path(".").rglob("*"):
    if any(skip in str(f) for skip in ["node_modules", ".git", "dist", "build", "__pycache__"]): continue
    if not f.is_file(): continue
    try:
        for i, line in enumerate(f.read_text(errors="replace").splitlines(), 1):
            for marker in MARKERS:
                if marker in line.upper():
                    found[marker].append(f"{f}:{i}  {line.strip()[:80]}")
    except: pass

total = sum(len(v) for v in found.values())
print(f"Total debt markers: {total}")
for marker, items in sorted(found.items(), key=lambda x: -len(x[1])):
    print(f"\n{marker} ({len(items)}):")
    for item in items[:5]: print(f"  {item}")
    if len(items) > 5: print(f"  ... and {len(items)-5} more")
EOF

# Estimate effort (rough heuristic: 1 TODO ≈ 30 min)
echo ""
echo "Rough effort estimate: $(grep -rn "TODO\|FIXME\|HACK" \
  --include="*.ts" --include="*.py" --include="*.go" \
  . | grep -v node_modules | wc -l) markers × 30min each"
```

**Tech debt report template:**
```
## Tech Debt Inventory — $(date +%Y-%m-%d)

| Category | Count | Est. Hours | Risk |
|----------|-------|-----------|------|
| TODO markers | N | N | Low |
| FIXME markers | N | N | Medium |
| HACK markers | N | N | High |
| Deprecated usage | N | N | High |
| Outdated deps (major ver) | N | N | Variable |
| Functions >40 lines | N | N | Medium |
| Files >500 lines | N | N | Medium |

**Total estimated debt:** N hours
```

---

## 2. Dead dependency removal

```bash
# Node.js — depcheck (unused deps)
npx depcheck 2>/dev/null | head -40

# Also check: packages used in only test/dev but listed in prod deps
cat package.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
deps = set(d.get('dependencies', {}).keys())
dev_deps = set(d.get('devDependencies', {}).keys())
print(f'Production deps: {len(deps)}')
print(f'Dev deps: {len(dev_deps)}')
# Look for test-only packages in prod deps
test_patterns = ['jest', 'vitest', 'mocha', 'chai', 'sinon', 'faker', 
                 'testing-library', 'playwright', 'cypress', 'supertest']
misplaced = [d for d in deps if any(p in d for p in test_patterns)]
if misplaced: print(f'Possible test deps in prod: {misplaced}')
"

# Python — deptry
deptry . 2>/dev/null | head -30
# or pip-check
pip-check 2>/dev/null | head -20

# Remove safely
npm uninstall <package-name> 2>/dev/null
# Always run tests after: npx jest
```

---

## 3. Unused environment variables

```bash
# Find all env vars referenced in code
grep -rn "process\.env\.\|os\.environ\.\|os\.getenv\|Config\." \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test\|spec" \
  | grep -oE "(process\.env\.[A-Z_]+|os\.environ\[['\"]([A-Z_]+)['\"]|os\.getenv\(['\"]([A-Z_]+)['\"])" \
  | sort -u > /tmp/code-envvars.txt

# Find all env vars defined in .env files
find . -name ".env*" -not -name "*.example" -not -name "*.sample" \
  | xargs grep -h "^[A-Z_]" 2>/dev/null \
  | cut -d= -f1 | sort -u > /tmp/defined-envvars.txt

# Compare
echo "=== Defined but never used ==="
comm -23 /tmp/defined-envvars.txt /tmp/code-envvars.txt

echo "=== Used but not defined (check .env.example) ==="
comm -13 /tmp/defined-envvars.txt /tmp/code-envvars.txt

# Verify .env.example is up to date
diff <(grep -oE "^[A-Z_]+" .env.example 2>/dev/null | sort) \
     <(grep -oE "^[A-Z_]+" .env 2>/dev/null | sort) 2>/dev/null
```

---

## 4. Log noise reduction

```bash
# Find all log statements
grep -rn "console\.log\|console\.debug\|console\.info\|print(\|logger\.debug\|logging\.debug" \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test\|spec" | wc -l

# Find debug logs that should be removed (not conditional)
grep -rn "console\.log\|console\.debug" \
  --include="*.ts" --include="*.js" \
  . | grep -v "node_modules\|test\|// " | head -30

# Find logs exposing sensitive data
grep -rn "console\.log\|print(\|logger\." \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -iv "node_modules" \
  | grep -iE "password|secret|token|key|credit|ssn|dob" | head -20

# Standardise to structured logging
cat > /tmp/logging-guide.md << 'EOF'
## Logging standards

REMOVE:  console.log() in production code (keep in scripts/tools only)
KEEP:    Structured logger calls with context

// BAD
console.log("User created:", user);
console.log("Error:", err);

// GOOD (structured, searchable, no sensitive data)
logger.info({ userId: user.id, action: "user.created" }, "User created");
logger.error({ err: { message: err.message, code: err.code } }, "User creation failed");

Levels:
  error  — something failed that needs human attention
  warn   — unexpected but handled; something to monitor
  info   — significant business events (order placed, user registered)
  debug  — developer diagnostics — MUST be disabled in production
EOF
cat /tmp/logging-guide.md
```

---

## 5. Error handling standardisation

```bash
# Find bare catch blocks (swallowing errors)
grep -rn "catch\s*(" --include="*.ts" --include="*.js" . \
  | grep -v "node_modules\|test" \
  > /tmp/catches.txt

# Empty or console-only catches
python3 - <<'EOF'
with open("/tmp/catches.txt") as f:
    lines = f.readlines()

for line in lines[:50]:
    print(line.rstrip())
EOF

# Python — bare except
grep -rn "except:" --include="*.py" . | grep -v "test\|venv" | head -20

# Find error handling inconsistencies
grep -rn "throw new Error\|throw new.*Error\|raise\|reject(" \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test" | head -30
```

**Error handling standardisation template:**
```typescript
// Create a central error hierarchy
// src/errors/index.ts

export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
    public readonly details?: unknown
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} not found: ${id}`, "NOT_FOUND", 404);
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public readonly fields: Record<string, string>) {
    super(message, "VALIDATION_ERROR", 400, { fields });
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = "Unauthorized") {
    super(message, "UNAUTHORIZED", 401);
  }
}

// Global error handler (Express)
export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: { code: err.code, message: err.message, details: err.details },
    });
  }
  // Unknown error — log and return generic message
  logger.error({ err }, "Unhandled error");
  return res.status(500).json({ error: { code: "INTERNAL_ERROR", message: "Something went wrong" } });
}
```

---

## 6. Legacy code modernisation

### Async/await migration (callbacks → promises)
```bash
# Find callback-style async patterns
grep -rn "function.*err.*callback\|\.then(.*\.catch\|new Promise" \
  --include="*.js" --include="*.ts" . | grep -v "node_modules\|test" | head -20
```

```javascript
// BEFORE — callback hell
function getUser(id, callback) {
  db.query("SELECT * FROM users WHERE id = ?", [id], (err, rows) => {
    if (err) return callback(err);
    callback(null, rows[0]);
  });
}

// AFTER — async/await
async function getUser(id: string): Promise<User | null> {
  const rows = await db.query("SELECT * FROM users WHERE id = ?", [id]);
  return rows[0] ?? null;
}
```

### var → const/let migration
```bash
grep -rn "\bvar\b" --include="*.js" --include="*.ts" \
  . | grep -v "node_modules" | wc -l

# Auto-fix with ESLint
npx eslint . --ext .js,.ts --rule '{"no-var": "error"}' --fix 2>/dev/null
```

### require → import/export (CommonJS → ESM)
```bash
grep -rn "require(" --include="*.ts" --include="*.js" . \
  | grep -v "node_modules\|// \|eslint" | head -20
```

```javascript
// BEFORE
const express = require("express");
const { join } = require("path");

// AFTER
import express from "express";
import { join } from "path";
```

### Python 2 → Python 3 patterns
```bash
grep -rn "print " --include="*.py" . | grep -v "test\|venv" | head -10
grep -rn "\.iteritems()\|\.itervalues()\|\.iterkeys()\|unicode(\|basestring" \
  --include="*.py" . | grep -v venv | head -20
```

---

## 7. Feature flag cleanup

```bash
# Find all feature flags
grep -rn "FEATURE_\|featureFlag\|feature_flag\|isFeatureEnabled\|flags\." \
  --include="*.ts" --include="*.js" --include="*.py" \
  . | grep -v "node_modules\|test\|spec" | head -30

# List flags with surrounding context to assess if they're dead
python3 - <<'EOF'
from pathlib import Path
import re

FLAG_PATTERN = re.compile(r"(FEATURE_[A-Z_]+|isFeatureEnabled\(['\"]([A-Z_a-z]+)['\"])")
found = {}

for f in Path(".").rglob("*.ts"):
    if "node_modules" in str(f): continue
    text = f.read_text(errors="replace")
    for match in FLAG_PATTERN.finditer(text):
        flag = match.group(1)
        found.setdefault(flag, []).append(str(f))

print(f"Feature flags found: {len(found)}")
for flag, files in sorted(found.items()):
    print(f"\n  {flag}")
    for file in set(files): print(f"    {file}")
EOF
```

**Cleanup process for a dead flag:**
1. Confirm the flag is `true` (always enabled) in all environments
2. Search all usages: `grep -rn "FLAG_NAME" .`
3. Remove the `if (flag)` wrapper — keep the truthy branch, delete the falsy branch
4. Remove the flag definition from config/feature-flag service
5. Run tests: `npx jest`
6. Commit: `git commit -m "chore: remove FLAG_NAME feature flag (fully rolled out)"`

---

## 8. i18n string extraction

```bash
# Find hardcoded user-facing strings in UI components
grep -rn ">[A-Z][a-zA-Z ]\{5,\}<\|placeholder=\"[A-Z]\|title=\"[A-Z]\|label=\"[A-Z]" \
  --include="*.tsx" --include="*.jsx" --include="*.vue" \
  . | grep -v "node_modules\|test" | head -30

# Check i18n coverage
npx i18n-ally . 2>/dev/null | head -20
```

**Extraction pattern (React + react-i18next):**
```tsx
// BEFORE (hardcoded)
<button>Save changes</button>
<p>Are you sure you want to delete this item?</p>

// AFTER (i18n)
import { useTranslation } from "react-i18next";

const { t } = useTranslation("common");
<button>{t("actions.save")}</button>
<p>{t("dialogs.delete.confirmation")}</p>
```

```json
// public/locales/en/common.json
{
  "actions": { "save": "Save changes" },
  "dialogs": { "delete": { "confirmation": "Are you sure you want to delete this item?" } }
}
```

---

## 9. Incremental refactor planning

For large refactors, always plan incrementally to keep `main` shippable:

```markdown
## Refactor Plan: [Name]

**Goal:** [What will be better after this refactor?]
**Risk level:** Low / Medium / High
**Estimated effort:** N days

### Phase 1 — Safe preparation (no behaviour change)
- [ ] Add missing tests for code to be refactored (get to 80% coverage)
- [ ] Extract magic numbers to named constants
- [ ] Fix any lint/type errors in scope

### Phase 2 — Structural change (behaviour-preserving)
- [ ] Extract helper functions (one function per commit)
- [ ] Rename for clarity (one file/module per commit)
- [ ] Move to correct layer/module

### Phase 3 — Logic change (if needed)
- [ ] Replace algorithm / pattern
- [ ] Update tests to cover new behaviour

### Phase 4 — Cleanup
- [ ] Remove old code / dead branches
- [ ] Update documentation

### Rollback plan
Each phase is a separate PR. Phase N can be reverted without affecting Phase N-1.

### Verification
After each commit: `npm test && npm run build`
After each PR: QA smoke test on staging
```

---

## 10. File size reduction

Large files are a smell — they usually do too many things:

```bash
# Find files over 300 lines
find . -name "*.ts" -o -name "*.py" -o -name "*.go" \
  | grep -v "node_modules\|.git\|dist\|build" \
  | xargs wc -l 2>/dev/null \
  | awk '$1 > 300 {print $1, $2}' \
  | sort -rn | head -20

# Suggest split points for large TypeScript files
python3 - <<'EOF'
from pathlib import Path
import re

for f in Path("src").rglob("*.ts"):
    lines = f.read_text(errors="replace").splitlines()
    if len(lines) < 300: continue
    
    exports = [(i+1, line.strip()) for i, line in enumerate(lines)
               if re.match(r"export (function|class|const|interface|type|enum)", line)]
    
    print(f"\n{f} ({len(lines)} lines) — {len(exports)} exports:")
    for lineno, decl in exports:
        print(f"  L{lineno:4d}  {decl[:60]}")
    if len(exports) > 3:
        print(f"  → Consider splitting by domain/responsibility")
EOF
```

---

## 11. Post-cleanup verification checklist

Run after every cleanup session:

```bash
# 1. Tests still pass
npx jest --passWithNoTests 2>/dev/null | tail -10
# or
pytest -q 2>/dev/null | tail -10

# 2. Build succeeds
npm run build 2>/dev/null | tail -10
# or
go build ./... 2>/dev/null

# 3. No new lint errors
npx eslint . --ext .ts,.tsx 2>/dev/null | tail -10

# 4. Type check passes
npx tsc --noEmit 2>/dev/null | tail -10

# 5. Bundle size didn't grow unexpectedly
du -sh dist/ build/ 2>/dev/null

# 6. Git diff sanity check
git diff --stat
git diff --name-only | wc -l
```

**Commit message conventions for cleanup:**
```
chore: remove unused UserHelper class
chore: extract validation logic to validators/user.ts
refactor: replace callback pattern with async/await in auth module
chore: remove FEATURE_NEW_DASHBOARD flag (rolled out)
chore: delete dead dependency 'lodash.get' (unused)
chore: extract hardcoded strings to en/common.json
```
