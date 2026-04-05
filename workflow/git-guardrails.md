---
name: git-guardrails
description: >
  Set up Claude Code hooks to block dangerous git commands and protect production
  branches. Prevents force pushes, accidental main branch commits, and destructive
  resets. Use when setting up a new project or when the user wants git safety rails.
  Triggers: "git guardrails", "protect main branch", "block force push", "git hooks",
  "prevent accidental", "safe git", "git safety".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Git Guardrails Skill

## What this sets up

1. Claude Code hook that blocks dangerous git commands before execution
2. Pre-commit hook that prevents direct commits to main/master
3. Confirmation prompts for destructive operations

---

## Step 1 — Create Claude Code hooks config

```bash
mkdir -p .claude/hooks
```

```json
// .claude/settings.json — add hooks section
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/check-git-command.sh"
          }
        ]
      }
    ]
  }
}
```

## Step 2 — Create the guard script

```bash
cat > .claude/hooks/check-git-command.sh << 'EOF'
#!/usr/bin/env bash
# Claude Code hook — checks bash commands before execution
# Reads the command from stdin as JSON

COMMAND=$(cat /dev/stdin | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('tool_input',{}).get('command',''))" 2>/dev/null)

# Block force push to any remote
if echo "$COMMAND" | grep -qE "git push.*(--force|-f)"; then
  echo "BLOCKED: Force push is not allowed. Use --force-with-lease if you must, after manual review."
  exit 1
fi

# Block hard reset
if echo "$COMMAND" | grep -qE "git reset --hard"; then
  echo "BLOCKED: Hard reset detected. Confirm you want to discard all local changes."
  exit 1
fi

# Block direct push to main/master
if echo "$COMMAND" | grep -qE "git push (origin )?main|git push (origin )?master"; then
  echo "BLOCKED: Direct push to main/master. Use a PR instead."
  exit 1
fi

# Block branch deletion of protected branches
if echo "$COMMAND" | grep -qE "git branch -[Dd] (main|master|develop|production|staging)"; then
  echo "BLOCKED: Cannot delete protected branch."
  exit 1
fi

# Block git clean -f (removes untracked files)
if echo "$COMMAND" | grep -qE "git clean -f"; then
  echo "BLOCKED: git clean -f will delete untracked files. Use -n (dry run) first."
  exit 1
fi

exit 0
EOF
chmod +x .claude/hooks/check-git-command.sh
```

## Step 3 — Pre-commit hook (Husky or native)

```bash
# Native git hook
mkdir -p .git/hooks
cat > .git/hooks/pre-commit << 'EOF'
#!/usr/bin/env bash
BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
PROTECTED="main master production staging"

for protected in $PROTECTED; do
  if [ "$BRANCH" = "$protected" ]; then
    echo "ERROR: Direct commits to '$BRANCH' are not allowed."
    echo "Create a feature branch: git checkout -b feat/your-feature"
    exit 1
  fi
done
EOF
chmod +x .git/hooks/pre-commit
echo "Pre-commit hook installed"
```

## Step 4 — Verify

```bash
# Test the guardrail
git checkout main
echo "test" > /tmp/test.txt
git add /tmp/test.txt 2>/dev/null || true
git commit -m "test commit" 2>/dev/null
# Expected: ERROR about protected branch
```

## Step 5 — Document in CONTRIBUTING.md

```bash
cat >> CONTRIBUTING.md << 'EOF'

## Branch protection
Direct commits to `main`, `master`, `production`, and `staging` are blocked.
Always work on a feature branch and open a PR.

Guardrails also block: `git push --force`, `git reset --hard`, `git clean -f`.
To bypass in an emergency, disable the pre-commit hook temporarily:
`HUSKY=0 git commit ...` or remove `.git/hooks/pre-commit` and re-add after.
EOF
```
