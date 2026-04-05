---
name: setup-pre-commit
description: >
  Set up Husky pre-commit hooks with lint-staged, Prettier formatting, TypeScript
  type checking, and test runs. Ensures every commit passes quality gates before
  it reaches the remote.
  Triggers: "set up pre-commit", "husky", "lint-staged", "commit hooks",
  "pre-commit hook", "format on commit", "type check on commit".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Setup Pre-commit Skill

## What gets installed
- **Husky** — git hooks manager
- **lint-staged** — runs linters only on staged files (fast)
- **Prettier** — code formatting
- **Type check** — TypeScript compile check on commit
- **Tests** — run affected tests before push

---

## Step 1 — Install dependencies

```bash
npm install -D husky lint-staged prettier 2>/dev/null
npx husky init 2>/dev/null
```

## Step 2 — Configure lint-staged

```json
// package.json — add lint-staged config
{
  "lint-staged": {
    "*.{ts,tsx,js,jsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yaml,yml,css}": [
      "prettier --write"
    ]
  }
}
```

## Step 3 — Pre-commit hook (fast — staged files only)

```bash
cat > .husky/pre-commit << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Run lint-staged (lint + format staged files only)
npx lint-staged

# TypeScript check (full project)
npx tsc --noEmit
EOF
chmod +x .husky/pre-commit
```

## Step 4 — Pre-push hook (tests before push)

```bash
cat > .husky/pre-push << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Run affected tests
npx jest --passWithNoTests --onlyFailures 2>/dev/null \
  || npx vitest run 2>/dev/null
EOF
chmod +x .husky/pre-push
```

## Step 5 — Prettier config

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

```
// .prettierignore
node_modules
dist
build
.next
coverage
*.generated.*
```

## Step 6 — Verify the setup

```bash
# Stage a file with formatting issues and commit
echo "const x =    'hello'" > /tmp/test-format.ts
git add /tmp/test-format.ts 2>/dev/null
git commit -m "test: pre-commit hook" 2>/dev/null
# Expected: Prettier reformats, ESLint checks, tsc checks — then commits (or fails on errors)

# Clean up
git reset HEAD /tmp/test-format.ts 2>/dev/null
rm /tmp/test-format.ts 2>/dev/null
```

## Skip hooks in emergency

```bash
# One-off skip (use sparingly)
git commit --no-verify -m "chore: emergency fix"
# Or
HUSKY=0 git push
```
