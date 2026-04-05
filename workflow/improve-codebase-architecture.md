---
name: improve-codebase-architecture
description: >
  Explore a codebase for architectural improvement opportunities, focusing on
  deepening shallow modules, improving testability, reducing coupling, and surfacing
  hidden complexity. Produces a prioritised improvement plan — not a rewrite.
  Triggers: "improve architecture", "codebase health", "architectural review",
  "shallow modules", "improve testability", "reduce coupling", "code health audit",
  "architecture improvements", "technical audit".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Improve Codebase Architecture Skill

## Philosophy

The goal is not a rewrite. The goal is targeted improvements that compound:
- **Deepen shallow modules** — modules that expose too much internal complexity
- **Improve testability** — extract logic that is hard to test in isolation
- **Reduce coupling** — break dependencies that make changes expensive
- **Clarify boundaries** — make it obvious what each module is responsible for

---

## Step 1 — Map the current architecture

```bash
# Directory shape
find src -type d | grep -v "node_modules\|.git\|dist" | sort

# Module sizes — outliers are God modules or dead weight
find src -name "*.ts" -o -name "*.py" -o -name "*.go" \
  | grep -v "node_modules\|test\|spec" \
  | xargs wc -l 2>/dev/null | sort -rn | head -20

# Dependency graph — detect heavy coupling hubs
npx madge --json src/ 2>/dev/null \
  | python3 -c "
import json,sys
from collections import Counter
data=json.load(sys.stdin)
inbound=Counter()
for src,deps in data.items():
    for d in deps: inbound[d]+=1
print('Most imported (high coupling risk):')
for f,c in inbound.most_common(15): print(f'  {c:3d}  {f}')
" 2>/dev/null

# Circular dependencies
npx madge --circular src/ 2>/dev/null
```

---

## Step 2 — Identify shallow modules

A shallow module has a complex interface relative to the implementation it hides.
Signs: large public API, many exported functions, classes with dozens of methods.

```bash
# Count exports per file — many exports = shallow/over-exposed
python3 - <<'EOF'
from pathlib import Path
import re

for f in sorted(Path("src").rglob("*.ts")):
    if "node_modules" in str(f) or "test" in str(f): continue
    text = f.read_text(errors="replace")
    exports = len(re.findall(r"^export\s+(function|class|const|interface|type|enum)", text, re.MULTILINE))
    lines   = len(text.splitlines())
    if exports > 8:
        ratio = round(lines / max(exports, 1))
        print(f"{exports:3d} exports  {lines:4d} lines  (~{ratio} lines/export)  {f}")
EOF
```

**What to do:** Group related exports behind a single façade interface. The caller
should know *what* the module does, not *how*.

---

## Step 3 — Identify testability problems

```bash
# Functions that reach directly into global/external state (hard to test)
grep -rn "process\.env\.\|fs\.read\|fetch(\|axios\.\|db\.\|prisma\." \
  --include="*.ts" src/ | grep -v "test\|spec\|config\|node_modules" \
  | grep -v "repository\|repo\|store\|client\." | head -20
# These should be behind injectable interfaces

# Classes with many constructor dependencies (hard to instantiate in tests)
python3 - <<'EOF'
from pathlib import Path
import re

for f in Path("src").rglob("*.ts"):
    if "test" in str(f) or "node_modules" in str(f): continue
    text = f.read_text(errors="replace")
    for m in re.finditer(r"constructor\(([^)]{80,})\)", text):
        params = m.group(1).count(",") + 1
        if params > 4:
            line = text[:m.start()].count("\n") + 1
            print(f"{params} deps  {f}:{line}")
EOF
```

**What to do:** Extract side-effectful operations into injected interfaces.
Pure functions are infinitely easier to test.

---

## Step 4 — Find hidden complexity

```bash
# High cyclomatic complexity — logic that needs decomposition
radon cc src -n C -s 2>/dev/null | head -20  # Python
npx eslint --rule '{"complexity":["warn",{"max":8}]}' \
  --ext .ts src/ 2>/dev/null | grep "complexity" | head -20

# Long functions — often hide multiple responsibilities
python3 - <<'EOF'
from pathlib import Path
import re

for f in Path("src").rglob("*.ts"):
    if "test" in str(f) or "node_modules" in str(f): continue
    lines = f.read_text(errors="replace").splitlines()
    depth, start, name = 0, 0, ""
    for i, line in enumerate(lines, 1):
        if re.search(r"(async\s+)?function\s+\w+|=>\s*{|\w+\s*=\s*async", line):
            start, name = i, line.strip()[:50]
        depth += line.count("{") - line.count("}")
        if start and depth <= 0:
            length = i - start
            if length > 40:
                print(f"{length:3d} lines  {f}:{start}  {name}")
            start = 0
EOF
```

---

## Step 5 — Produce improvement plan

Output prioritised improvements in this format:

```markdown
## Architecture Improvement Plan

### Priority 1 — High impact, low risk (do this sprint)

**1. Extract [X] interface from [Module]**
- Problem: [Module] reaches directly into the database, making it impossible to unit test
- Fix: Extract `IUserRepository` interface; inject via constructor
- Files: `src/users/UserService.ts` → add interface + DI
- Effort: 2h  Risk: Low (behaviour-preserving)

**2. Deepen [Y] module**
- Problem: `src/utils/index.ts` exports 23 functions — callers are coupled to implementation details
- Fix: Group into 3 domain-specific modules with narrow public APIs
- Effort: 3h  Risk: Low (rename + move, no logic change)

### Priority 2 — Medium impact, medium risk (next sprint)

**3. Break circular dependency [A] ↔ [B]**
- Problem: `src/orders` imports from `src/users` and vice versa
- Fix: Extract shared `types.ts` that both import from
- Effort: 4h  Risk: Medium (touches import graph)

### Priority 3 — Backlog

**4. Decompose [God class]**
...

### What NOT to rewrite
- [List things that look messy but are stable and tested — leave them alone]
```

---

## Step 6 — Execute Phase 1 only

Start with the highest-impact, lowest-risk improvement. Verify tests pass after
each step. Never combine multiple improvements in one commit.

```bash
npm test 2>/dev/null | tail -5   # green before
# make one improvement
npm test 2>/dev/null | tail -5   # green after
git commit -m "refactor: extract IUserRepository interface for testability"
```
