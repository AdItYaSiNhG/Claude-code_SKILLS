---
name: code-quality
description: >
  Use this skill for code quality and compliance tasks: enforcing coding standards
  (PEP8, Airbnb, Google style), FOSS license audits, dead code removal, cyclomatic
  complexity reduction, coupling/cohesion analysis, code smell detection, docstring
  standards, naming convention enforcement, static type checking, and accessibility
  compliance (WCAG).
  Triggers: "code quality", "lint", "style guide", "license audit", "dead code",
  "complexity", "code smell", "type check", "naming conventions", "WCAG", "a11y".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Code Quality & Compliance Skill

## Why this skill exists

Quality issues compound. A codebase with inconsistent style, unchecked complexity,
undocumented functions, and ignored lint warnings is expensive to maintain and
dangerous to extend. This skill gives Claude a **systematic, tool-backed quality
review** across all dimensions — style, types, complexity, licensing, and accessibility.

Always produce an **actionable finding list with file:line references** and
auto-fix commands wherever tools support them.

---

## 1. Language & toolchain detection

```bash
# Identify what quality tools are already configured
ls -la .eslintrc* .eslintignore .prettierrc* \
  pyproject.toml setup.cfg .flake8 .mypy.ini \
  .golangci.yml .rubocop.yml biome.json 2>/dev/null

# Check existing scripts
cat package.json 2>/dev/null \
  | python3 -c "import json,sys; d=json.load(sys.stdin); \
    [print(k,':',v) for k,v in d.get('scripts',{}).items() \
     if any(w in v for w in ['lint','format','type','check'])]"
```

---

## 2. Linting & style enforcement

### JavaScript / TypeScript — ESLint + Prettier
```bash
# Run ESLint (output as JSON for parsing)
npx eslint . --ext .js,.jsx,.ts,.tsx --format json \
  --output-file /tmp/eslint-report.json 2>/dev/null

# Summarise findings by rule
cat /tmp/eslint-report.json | python3 -c "
import json, sys
from collections import Counter
data = json.load(sys.stdin)
rules = Counter()
errors, warnings = 0, 0
for f in data:
    for m in f.get('messages', []):
        rules[m.get('ruleId','unknown')] += 1
        if m['severity'] == 2: errors += 1
        else: warnings += 1
print(f'Errors: {errors}  Warnings: {warnings}')
print()
for rule, count in rules.most_common(20):
    print(f'  {count:4d}  {rule}')
"

# Auto-fix safe issues
npx eslint . --ext .js,.jsx,.ts,.tsx --fix 2>/dev/null

# Prettier check (no auto-fix yet — show diff first)
npx prettier --check "**/*.{ts,tsx,js,jsx,json,css,md}" 2>/dev/null | tail -20
# Auto-format
npx prettier --write "**/*.{ts,tsx,js,jsx,json,css,md}" 2>/dev/null
```

**Recommended ESLint config additions for enterprise:**
```json
{
  "rules": {
    "no-console": "warn",
    "no-debugger": "error",
    "eqeqeq": ["error", "always"],
    "no-var": "error",
    "prefer-const": "error",
    "no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/no-explicit-any": "warn",
    "@typescript-eslint/explicit-function-return-type": "warn"
  }
}
```

### Python — Ruff + Black + isort
```bash
# Ruff (fast, replaces flake8 + isort + many plugins)
ruff check . --output-format json 2>/dev/null \
  | python3 -c "
import json, sys
from collections import Counter
data = json.load(sys.stdin)
codes = Counter(d.get('code','?') for d in data)
print(f'Total findings: {len(data)}')
for code, count in codes.most_common(15):
    print(f'  {count:4d}  {code}')
"

# Auto-fix
ruff check . --fix 2>/dev/null
ruff format . 2>/dev/null   # replaces black

# Fallback: flake8 + black + isort
flake8 . --max-line-length 88 --extend-ignore E203 \
  --exclude venv,.venv,migrations --statistics 2>/dev/null | tail -20
black --check --diff . 2>/dev/null | head -50
isort --check-only --diff . 2>/dev/null | head -30
```

### Go
```bash
gofmt -l . 2>/dev/null | head -20          # files needing formatting
goimports -l . 2>/dev/null | head -20       # imports order
golangci-lint run --out-format json 2>/dev/null \
  | python3 -c "
import json, sys
from collections import Counter
d = json.load(sys.stdin)
issues = d.get('Issues', [])
linters = Counter(i.get('FromLinter','?') for i in issues)
print(f'Total: {len(issues)}')
for l, c in linters.most_common(10): print(f'  {c:4d}  {l}')
" 2>/dev/null
```

### Java / Kotlin
```bash
# Checkstyle
./mvnw checkstyle:check 2>/dev/null | grep "\[WARN\]\|\[ERROR\]" | head -30
# ktlint (Kotlin)
ktlint --reporter=plain "**/*.kt" 2>/dev/null | head -30
```

### Rust
```bash
cargo clippy -- -D warnings 2>/dev/null | head -50
cargo fmt --check 2>/dev/null
```

---

## 3. Static type checking

### TypeScript
```bash
npx tsc --noEmit --strict 2>/dev/null \
  | head -50

# Count errors by category
npx tsc --noEmit --strict 2>/dev/null \
  | grep "error TS" \
  | sed 's/.*error TS\([0-9]*\).*/TS\1/' \
  | sort | uniq -c | sort -rn | head -20
```

### Python — mypy + pyright
```bash
mypy . --ignore-missing-imports --strict 2>/dev/null \
  | tail -30

# Pyright (VS Code compatible)
pyright . 2>/dev/null | tail -20

# Count by error type
mypy . --ignore-missing-imports 2>/dev/null \
  | grep "error:" \
  | sed 's/.*error: //' \
  | sort | uniq -c | sort -rn | head -15
```

---

## 4. Cyclomatic complexity analysis

High complexity (>10) makes code hard to test and maintain.

```bash
# JavaScript / TypeScript — complexity-report or eslint rule
npx eslint . --ext .ts,.js \
  --rule '{"complexity": ["warn", {"max": 10}]}' \
  --format compact 2>/dev/null | grep "complexity" | head -30

# Python — radon
radon cc . -n C -s 2>/dev/null \
  | head -30   # shows only C (complex, 11-15) and above

# Full report sorted by complexity
radon cc . -s --json 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
funcs = []
for path, items in data.items():
    for item in items:
        funcs.append((item['complexity'], path, item['name'], item['lineno']))
for c, p, n, l in sorted(funcs, reverse=True)[:20]:
    grade = 'A' if c<=5 else 'B' if c<=10 else 'C' if c<=15 else 'D' if c<=20 else 'F'
    print(f'{grade} ({c:3d})  {p}:{l}  {n}')
"

# Go — gocognit
gocognit -top 20 ./... 2>/dev/null

# Java — PMD
pmd check -d src -R rulesets/java/quickstart.xml -f text 2>/dev/null | head -30
```

**Complexity thresholds:**
| Score | Grade | Action |
|-------|-------|--------|
| 1–5 | A | Good |
| 6–10 | B | Acceptable |
| 11–15 | C | Refactor soon |
| 16–20 | D | Refactor this sprint |
| 21+ | F | Critical — break up now |

---

## 5. Dead code detection

```bash
# TypeScript / JavaScript — ts-prune or knip
npx knip 2>/dev/null | head -50    # exports, deps, files that aren't used
npx ts-prune 2>/dev/null | head -30

# Python — vulture
vulture . --min-confidence 80 2>/dev/null | head -30

# Go
deadcode ./... 2>/dev/null | head -30

# Generic — find files never imported/required
# (works for any language — adapt extension)
python3 - <<'EOF'
import os, re, sys
from pathlib import Path

EXT = ".ts"   # change as needed
root = Path(".")
all_files = [p for p in root.rglob(f"*{EXT}") if "node_modules" not in str(p)]
source = ""
for f in all_files:
    try: source += f.read_text(errors="replace")
    except: pass

unused = []
for f in all_files:
    stem = f.stem
    # skip index files, they're re-exports
    if stem in ("index", "main", "app", "server"): continue
    if not re.search(rf"['\"].*{re.escape(stem)}['\"]|from.*{re.escape(stem)}", source.replace(f.read_text(errors='replace'), "")):
        unused.append(str(f))

print(f"Potentially unused {EXT} files ({len(unused)}):")
for u in unused[:30]: print(f"  {u}")
EOF
```

---

## 6. Code smell detection

```bash
# Duplicate code — jscpd
npx jscpd . --threshold 5 --ignore "node_modules,dist,build" \
  --reporters console 2>/dev/null | tail -40

# Long functions / files
python3 - <<'EOF'
from pathlib import Path
import re

LANG_CONFIGS = {
    "*.ts": (50, r"(function|=>|async\s+\w+\s*\()"),
    "*.py": (40, r"def "),
    "*.go": (40, r"func "),
}

for glob, (threshold, func_re) in LANG_CONFIGS.items():
    for f in Path(".").rglob(glob):
        if "node_modules" in str(f) or ".git" in str(f): continue
        lines = f.read_text(errors="replace").splitlines()
        if len(lines) > 500:
            print(f"LONG FILE ({len(lines)} lines): {f}")
        
        # find long functions
        func_start = None
        func_name = ""
        depth = 0
        for i, line in enumerate(lines, 1):
            if re.search(func_re, line): func_start, func_name = i, line.strip()[:60]
            depth += line.count("{") - line.count("}")
            if func_start and depth <= 0:
                length = i - func_start
                if length > threshold:
                    print(f"LONG FN ({length}L): {f}:{func_start} {func_name}")
                func_start = None
EOF

# God class / large class detection (Python)
radon mi . -n B 2>/dev/null | head -20  # maintainability index
```

---

## 7. Naming convention audit

```bash
python3 - <<'EOF'
from pathlib import Path
import re, sys

issues = []

for f in Path(".").rglob("*.ts"):
    if "node_modules" in str(f) or ".git" in str(f): continue
    text = f.read_text(errors="replace")
    lines = text.splitlines()
    for i, line in enumerate(lines, 1):
        # snake_case variable names in TS (should be camelCase)
        m = re.search(r"\bconst\s+([a-z]+_[a-z_]+)\s*=", line)
        if m and not m.group(1).startswith("_"):
            issues.append(f"snake_case const in TS: {f}:{i}  {m.group(1)}")
        # ALL_CAPS non-const usage (should only be for true constants)
        m = re.search(r"\b(let|var)\s+([A-Z_]{3,})\b", line)
        if m:
            issues.append(f"ALL_CAPS with let/var: {f}:{i}  {m.group(2)}")

for f in Path(".").rglob("*.py"):
    if "venv" in str(f) or ".git" in str(f): continue
    text = f.read_text(errors="replace")
    for i, line in enumerate(text.splitlines(), 1):
        # CamelCase function (should be snake_case in Python)
        m = re.search(r"def ([A-Z][a-z]+[A-Z]\w*)\(", line)
        if m:
            issues.append(f"CamelCase function in Python: {f}:{i}  {m.group(1)}")

for issue in issues[:40]:
    print(issue)
print(f"\nTotal: {len(issues)} naming issues")
EOF
```

---

## 8. Docstring & comment standards

```bash
# Python — pydocstyle / interrogate
interrogate . --ignore-init-method --ignore-magic-methods \
  --fail-under 80 -v 2>/dev/null | tail -20

pydocstyle . --convention=google 2>/dev/null | head -40

# TypeScript — check exported functions/classes for JSDoc
python3 - <<'EOF'
from pathlib import Path
import re

missing = []
for f in Path(".").rglob("*.ts"):
    if "node_modules" in str(f): continue
    lines = f.read_text(errors="replace").splitlines()
    for i, line in enumerate(lines):
        is_export = re.match(r"\s*export\s+(async\s+)?function|export\s+(default\s+)?class", line)
        if is_export:
            # look back for JSDoc
            prev = lines[max(0,i-3):i]
            if not any("*/" in p or "/**" in p for p in prev):
                missing.append(f"{f}:{i+1}  {line.strip()[:60]}")

print(f"Exported symbols missing JSDoc ({len(missing)}):")
for m in missing[:30]: print(f"  {m}")
EOF
```

---

## 9. FOSS license compliance audit

```bash
# Node.js — license-checker
npx license-checker --json --out /tmp/licenses.json 2>/dev/null
cat /tmp/licenses.json | python3 -c "
import json, sys
from collections import Counter

data = json.load(sys.stdin)
lics = Counter()
risky = []
RISKY = {'GPL-2.0', 'GPL-3.0', 'AGPL-3.0', 'LGPL-2.1', 'LGPL-3.0', 'CC-BY-NC', 'UNLICENSED'}
for pkg, info in data.items():
    lic = info.get('licenses', 'Unknown')
    lics[lic] += 1
    if any(r in str(lic) for r in RISKY):
        risky.append(f'{pkg}: {lic}')

print('License breakdown:')
for l, c in lics.most_common(): print(f'  {c:4d}  {l}')
print(f'\nRisky licenses ({len(risky)}):')
for r in risky: print(f'  {r}')
"

# Python — pip-licenses
pip-licenses --format=json 2>/dev/null | python3 -c "
import json, sys
data = json.load(sys.stdin)
RISKY = {'GPL', 'AGPL', 'LGPL', 'CC-BY-NC', 'UNKNOWN'}
risky = [d for d in data if any(r in d.get('License','') for r in RISKY)]
print(f'Total packages: {len(data)}, Risky: {len(risky)}')
for r in risky: print(f\"  {r['Name']} {r['Version']}: {r['License']}\")
"

# Go
go-licenses report ./... 2>/dev/null | grep -i "gpl\|agpl\|lgpl" | head -20
```

**License risk matrix:**
| Risk | Licenses | Implication |
|------|---------|-------------|
| High | AGPL-3.0 | Must open-source your app |
| High | GPL-2.0 / GPL-3.0 | Copyleft — infects combined work |
| Medium | LGPL-2.1 / LGPL-3.0 | Dynamic linking usually OK |
| Low | MIT / Apache-2.0 / BSD | Permissive — safe for commercial use |
| Unknown | UNLICENSED | No explicit grant — legally risky |

---

## 10. Accessibility compliance (WCAG 2.1 AA)

```bash
# axe-core CLI (requires Node)
npx @axe-core/cli http://localhost:3000 --reporter json 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
violations = data.get('violations', [])
print(f'Violations: {len(violations)}')
for v in sorted(violations, key=lambda x: x.get('impact',''), reverse=True):
    print(f\"  [{v.get('impact','?').upper():8}] {v['id']}: {v['description'][:70]}\")
    for node in v.get('nodes', [])[:2]:
        print(f\"    Target: {node.get('target', '')}\")
"

# pa11y — page-level audit
npx pa11y http://localhost:3000 --standard WCAG2AA --reporter json 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
issues = data if isinstance(data, list) else data.get('issues', [])
from collections import Counter
types = Counter(i.get('type','?') for i in issues)
print(f'Total issues: {len(issues)}')
for t, c in types.items(): print(f'  {t}: {c}')
"

# Static HTML checks — no-alt img, missing labels
grep -rn "<img" --include="*.html" --include="*.tsx" --include="*.jsx" . \
  | grep -v "alt=" | grep -v "node_modules" | head -20

grep -rn "<input" --include="*.html" --include="*.tsx" --include="*.jsx" . \
  | grep -v "node_modules" > /tmp/inputs.txt
grep -rn "<label" --include="*.html" --include="*.tsx" --include="*.jsx" . \
  | grep -v "node_modules" > /tmp/labels.txt
echo "Inputs: $(wc -l < /tmp/inputs.txt)  Labels: $(wc -l < /tmp/labels.txt)"
```

---

## 11. Quality report template

```
## Code Quality Report
Date: $(date)
Scope: [repo]

### Linting
- ESLint errors: N  warnings: N
- Top rules violated: [list]

### Type Safety
- TypeScript errors: N
- any usages: N  (target: 0)

### Complexity
- Functions with complexity > 10: N
- Worst offenders: [file:line (score)]

### Dead Code
- Unused exports: N
- Unused dependencies: [list]

### Documentation
- Docstring coverage: N%  (target: 80%)

### License Risk
- High-risk licenses: N packages
- Packages requiring review: [list]

### Accessibility
- WCAG AA violations: N
- Critical (must-fix): [list]

### Recommended actions (this sprint)
1. ...
2. ...

### Recommended actions (backlog)
1. ...
```

---

## Auto-fix command cheatsheet

```bash
# JS/TS — fix all safe lint issues + format
npx eslint . --ext .ts,.tsx,.js,.jsx --fix && npx prettier --write .

# Python — fix lint + format + sort imports
ruff check . --fix && ruff format .

# Go — format
gofmt -w . && goimports -w .

# Rust
cargo fmt && cargo clippy --fix --allow-dirty

# Java (spotless)
./mvnw spotless:apply
```
