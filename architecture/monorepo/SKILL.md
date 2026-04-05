---
name: monorepo
description: >
  Use this skill for all monorepo tasks: Nx and Turborepo setup and configuration,
  workspace dependency graphs, package versioning with Changesets, build caching
  strategy, cross-app type sharing, shared config packages, affected-build pipelines,
  and workspace tooling. Also covers polyrepo-to-monorepo migration.
  Triggers: "monorepo", "Nx", "Turborepo", "workspace", "pnpm workspace",
  "Changesets", "affected", "build cache", "cross-app types", "shared packages",
  "package versioning", "lerna", "polyrepo to monorepo".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Monorepo Skill

## Why this skill exists

Monorepos done wrong become slower and harder to maintain than the polyrepos
they replaced. This skill ensures monorepos are set up with **correct build
caching, clean dependency graphs, automated versioning, and CI pipelines
that only rebuild what changed** — so scale doesn't kill developer velocity.

---

## 0. Audit existing monorepo (if one exists)

```bash
# Detect monorepo tooling
cat package.json 2>/dev/null | python3 -c "
import json, sys
d = json.load(sys.stdin)
tools = []
all_deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
if 'nx' in all_deps: tools.append('Nx')
if 'turbo' in all_deps or 'turborepo' in all_deps: tools.append('Turborepo')
if 'lerna' in all_deps: tools.append('Lerna (legacy)')
if d.get('workspaces'): tools.append(f\"npm/yarn workspaces: {d['workspaces']}\")
print('Monorepo tools:', tools or ['none detected'])
"

cat pnpm-workspace.yaml 2>/dev/null && echo "pnpm workspace found"
ls nx.json turbo.json lerna.json 2>/dev/null

# List all packages/apps
find . -name "package.json" \
  | grep -v "node_modules\|dist\|build\|coverage" \
  | xargs grep -l '"name"' \
  | while read f; do
    name=$(python3 -c "import json; d=json.load(open('$f')); print(d.get('name','?'))")
    echo "$name  ($f)"
  done | sort

# Package dependency graph
npx nx graph 2>/dev/null &   # opens browser
turbo run build --dry=json 2>/dev/null | python3 -m json.tool | head -60
```

---

## 1. Turborepo — setup and configuration

### pnpm workspace monorepo with Turborepo

**Directory structure:**
```
myapp/
├── apps/
│   ├── web/                    # Next.js frontend
│   ├── api/                    # Express/Fastify backend
│   └── admin/                  # Admin panel
│
├── packages/
│   ├── ui/                     # shared React component library
│   ├── types/                  # shared TypeScript types
│   ├── utils/                  # shared utility functions
│   ├── config/                 # shared configs (tsconfig, eslint, jest)
│   │   ├── tsconfig/
│   │   ├── eslint-config/
│   │   └── jest-config/
│   └── database/               # Prisma schema + client (shared)
│
├── turbo.json                  # pipeline config
├── pnpm-workspace.yaml
└── package.json                # root — dev tooling only
```

**pnpm-workspace.yaml:**
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

**turbo.json — full pipeline:**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "ui": "tui",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["$TURBO_DEFAULT$", ".env*"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**", "build/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "inputs": ["$TURBO_DEFAULT$", "tests/**"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "lint": {
      "dependsOn": ["^lint"],
      "inputs": ["$TURBO_DEFAULT$"],
      "outputs": []
    },
    "type-check": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "clean": {
      "cache": false
    }
  },
  "remoteCache": {
    "enabled": true
  }
}
```

**Root package.json:**
```json
{
  "name": "myapp-monorepo",
  "private": true,
  "scripts": {
    "build":      "turbo run build",
    "test":       "turbo run test",
    "lint":       "turbo run lint",
    "type-check": "turbo run type-check",
    "dev":        "turbo run dev",
    "clean":      "turbo run clean && rimraf node_modules",
    "changeset":  "changeset",
    "version":    "changeset version",
    "release":    "turbo run build && changeset publish"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "@changesets/cli": "^2.27.0",
    "rimraf": "^5.0.0",
    "typescript": "^5.4.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

---

## 2. Nx — setup and configuration

### Nx workspace structure
```
myapp/
├── apps/
│   ├── web/                    # application
│   │   ├── src/
│   │   ├── project.json        # Nx project config
│   │   └── tsconfig.app.json
│   └── api/
│       ├── src/
│       └── project.json
│
├── libs/                       # Nx convention: packages → libs
│   ├── shared/
│   │   ├── ui/
│   │   │   ├── src/
│   │   │   ├── project.json
│   │   │   └── index.ts        # public API
│   │   ├── types/
│   │   └── utils/
│   └── feature/
│       ├── auth/
│       └── checkout/
│
├── nx.json                     # workspace config
├── tsconfig.base.json          # path mappings
└── package.json
```

**nx.json:**
```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.ts",
      "!{projectRoot}/jest.config.ts",
      "!{projectRoot}/.eslintrc.json"
    ],
    "sharedGlobals": []
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"],
      "cache": true
    },
    "test": {
      "inputs": ["default", "^production"],
      "cache": true
    },
    "lint": {
      "inputs": ["default"],
      "cache": true
    }
  },
  "plugins": [
    "@nx/eslint/plugin",
    "@nx/jest/plugin",
    "@nx/next/plugin",
    "@nx/node/plugin"
  ]
}
```

**tsconfig.base.json — path mappings for all libs:**
```json
{
  "compilerOptions": {
    "paths": {
      "@myapp/ui":              ["libs/shared/ui/src/index.ts"],
      "@myapp/types":           ["libs/shared/types/src/index.ts"],
      "@myapp/utils":           ["libs/shared/utils/src/index.ts"],
      "@myapp/feature-auth":    ["libs/feature/auth/src/index.ts"],
      "@myapp/feature-checkout":["libs/feature/checkout/src/index.ts"]
    }
  }
}
```

**Run only affected projects (key CI optimisation):**
```bash
# Compare against main branch
npx nx affected --target=build --base=main --head=HEAD
npx nx affected --target=test  --base=main --head=HEAD
npx nx affected --target=lint  --base=main --head=HEAD

# Visualise what's affected
npx nx graph --affected --base=main

# Run all targets for affected projects in parallel
npx nx affected -t build test lint --parallel=5
```

---

## 3. Package scaffold templates

### Shared UI component library
```
packages/ui/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.test.tsx
│   │   │   └── Button.stories.tsx    # Storybook
│   │   ├── Input/
│   │   └── Modal/
│   └── index.ts                      # exports ONLY public components
├── package.json
├── tsconfig.json
└── vite.config.ts                    # builds ESM + CJS dual output
```

```json
// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "0.0.0",
  "private": false,
  "type": "module",
  "exports": {
    ".": {
      "import":  "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types":   "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "vite build && tsc --emitDeclarationOnly",
    "dev":   "vite build --watch",
    "test":  "vitest run",
    "lint":  "eslint src --ext .ts,.tsx"
  },
  "peerDependencies": {
    "react": ">=18",
    "react-dom": ">=18"
  },
  "devDependencies": {
    "@myapp/config": "workspace:*",
    "react": "^18",
    "react-dom": "^18",
    "typescript": "^5.4.0",
    "vite": "^5.0.0"
  }
}
```

### Shared types package
```
packages/types/
├── src/
│   ├── api/
│   │   ├── users.ts           # User, CreateUserDto, UserResponse
│   │   ├── orders.ts
│   │   └── index.ts
│   ├── domain/
│   │   ├── enums.ts           # UserRole, OrderStatus, etc.
│   │   └── index.ts
│   └── index.ts               # re-exports everything
└── package.json
```

```json
// packages/types/package.json — types-only, no build step needed
{
  "name": "@myapp/types",
  "version": "0.0.0",
  "private": false,
  "main":  "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "type-check": "tsc --noEmit",
    "lint": "eslint src --ext .ts"
  },
  "devDependencies": {
    "@myapp/config": "workspace:*",
    "typescript": "^5.4.0"
  }
}
```

---

## 4. Changesets — package versioning

### Setup
```bash
pnpm add -D @changesets/cli -w    # install at workspace root
pnpm changeset init               # creates .changeset/config.json
```

**.changeset/config.json:**
```json
{
  "$schema": "https://unpkg.com/@changesets/config/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [["@myapp/web", "@myapp/admin"]],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": ["@myapp/config"]
}
```

### Release workflow

**Developer (after making a change to a package):**
```bash
pnpm changeset        # interactive — pick packages changed + bump type
# creates .changeset/purple-lions-dance.md

git add .changeset/
git commit -m "chore: add changeset for ui button fix"
```

**CI release pipeline:**
```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile

      - name: Create Release PR or publish
        uses: changesets/action@v1
        with:
          publish: pnpm release      # runs build + changeset publish
          version: pnpm version      # runs changeset version
          commit: "chore: version packages"
          title: "chore: version packages"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 5. Build caching strategy

### Turborepo remote cache (Vercel)
```bash
# Authenticate (Vercel account)
npx turbo login
npx turbo link    # link to Vercel remote cache

# GitHub Actions — cache hit rate check
turbo run build --dry=json 2>/dev/null \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
tasks = d.get('tasks', [])
cached = sum(1 for t in tasks if t.get('cache',{}).get('status') == 'HIT')
print(f'Cache hit rate: {cached}/{len(tasks)} ({100*cached//max(len(tasks),1)}%)')
"
```

### Self-hosted Turborepo cache (open source)
```bash
# Deploy ducktape/turborepo-remote-cache or use S3
docker run -d -p 3001:3001 \
  -e PORT=3001 \
  -e STORAGE_PROVIDER=s3 \
  -e STORAGE_PATH=s3://my-turbo-cache \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  ducktape/turborepo-remote-cache

# Use in CI
turbo run build --api=http://cache-server:3001 --token=$TURBO_TOKEN
```

### Nx Cloud remote cache
```bash
# Free tier available
npx nx connect
# or self-host
npx nx g @nx/workspace:activate-cloud
```

---

## 6. Workspace dependency graph analysis

```bash
# Find internal dependency cycles
npx nx graph --file=output.json 2>/dev/null \
  && cat output.json | python3 -c "
import json, sys

data = json.load(sys.stdin)
graph = data.get('graph', {})
nodes = graph.get('nodes', {})
deps = graph.get('dependencies', {})

print(f'Packages: {len(nodes)}')
print(f'\nDependency counts (most depended-on = high coupling):')
inbound = {}
for pkg, pkg_deps in deps.items():
    for d in pkg_deps:
        target = d.get('target','')
        inbound[target] = inbound.get(target, 0) + 1

for pkg, count in sorted(inbound.items(), key=lambda x: -x[1])[:15]:
    print(f'  {count:3d}  {pkg}')
" 2>/dev/null

# Detect packages with too many dependents (change risk)
python3 - <<'EOF'
from pathlib import Path
import json, re

# Scan all package.json files for workspace dependencies
inbound = {}
packages = {}

for f in Path(".").rglob("package.json"):
    if "node_modules" in str(f) or f.parent == Path("."): continue
    try:
        d = json.loads(f.read_text())
        name = d.get("name")
        if not name: continue
        packages[name] = str(f.parent)
        all_deps = {**d.get("dependencies",{}), **d.get("devDependencies",{})}
        for dep, ver in all_deps.items():
            if "workspace:" in str(ver):
                inbound[dep] = inbound.get(dep, 0) + 1
    except: pass

print("Internal package dependency count:")
for pkg, count in sorted(inbound.items(), key=lambda x: -x[1]):
    flag = " ⚠️  high coupling" if count >= 4 else ""
    print(f"  {count:3d}  {pkg}{flag}")
EOF
```

---

## 7. Affected-build CI pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml — only tests what changed
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      affected: ${{ steps.affected.outputs.affected }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - id: affected
        run: |
          # Turborepo: output affected packages as JSON
          echo "affected=$(turbo run build --dry=json 2>/dev/null \
            | python3 -c \"import json,sys; \
              d=json.load(sys.stdin); \
              pkgs=list({t['package'] for t in d.get('tasks',[]) \
                         if t.get('package')}); \
              print(json.dumps(pkgs))\")" >> $GITHUB_OUTPUT

  build-and-test:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile

      # Turborepo only runs tasks for changed packages (cache hits for rest)
      - name: Build affected
        run: turbo run build --filter="...[origin/main]"
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM:  ${{ vars.TURBO_TEAM }}

      - name: Test affected
        run: turbo run test --filter="...[origin/main]" -- --coverage

      - name: Lint affected
        run: turbo run lint --filter="...[origin/main]"
```

---

## 8. Monorepo health checks

```bash
# 1. Mismatched dependency versions (same package, different versions)
python3 - <<'EOF'
from pathlib import Path
import json
from collections import defaultdict

versions = defaultdict(dict)

for f in Path(".").rglob("package.json"):
    if "node_modules" in str(f): continue
    try:
        d = json.loads(f.read_text())
        name = d.get("name", str(f.parent))
        all_deps = {**d.get("dependencies",{}), **d.get("devDependencies",{})}
        for dep, ver in all_deps.items():
            if not dep.startswith("@myapp"): # skip internal
                versions[dep][name] = ver
    except: pass

mismatches = {dep: vers for dep, vers in versions.items() if len(set(vers.values())) > 1}
print(f"Dependency version mismatches ({len(mismatches)}):")
for dep, vers in sorted(mismatches.items()):
    print(f"\n  {dep}:")
    for pkg, ver in vers.items():
        print(f"    {ver:20s}  {pkg}")
EOF

# 2. Missing peerDependencies
for pkg in packages/*/package.json; do
  echo "=== $pkg ==="
  python3 -c "
import json
d = json.load(open('$pkg'))
peer = set(d.get('peerDependencies',{}).keys())
dev = set(d.get('devDependencies',{}).keys())
missing = peer - dev
if missing: print(f'peerDep not in devDep: {missing}')
  "
done

# 3. Packages with no tests
for pkg_dir in apps/* packages/*; do
  test_count=$(find "$pkg_dir" -name "*.test.ts" -o -name "*.spec.ts" 2>/dev/null | wc -l)
  if [ "$test_count" -eq 0 ]; then
    echo "NO TESTS: $pkg_dir"
  fi
done

# 4. Circular workspace dependencies
npx madge --circular --extensions ts packages/ apps/ 2>/dev/null | head -20
```

---

## 9. Polyrepo → monorepo migration plan

```bash
# Step 1 — inventory all repos to migrate
# List each repo: name, language, dependencies on other repos, team owner

# Step 2 — set up monorepo skeleton (no code yet)
mkdir myapp-monorepo && cd myapp-monorepo
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - "apps/*"
  - "packages/*"
EOF
pnpm init
pnpm add -D turbo typescript @changesets/cli -w

# Step 3 — migrate repos one at a time (least-depended-on first)
# Clone into the right path
git clone https://github.com/myorg/ui-library packages/ui
cd packages/ui
# Update package.json: set name to @myapp/ui, version to 0.0.0

# Step 4 — replace cross-repo dependencies with workspace:*
# In each package.json:
# BEFORE: "@myorg/ui": "^1.2.3"   (npm version)
# AFTER:  "@myapp/ui": "workspace:*"

# Step 5 — run the full pipeline and verify
pnpm install
turbo run build
turbo run test
```

---

## 10. Workspace scripts cheatsheet

```bash
# Run a command in a specific package
pnpm --filter @myapp/web dev
pnpm --filter @myapp/ui build
turbo run test --filter=@myapp/api

# Run in all packages matching a pattern
pnpm --filter "@myapp/*" build
turbo run lint --filter="apps/*"

# Run only packages affected by changes vs main
turbo run test --filter="...[origin/main]"
pnpm --filter "...[origin/main]" test

# Add a dependency to a specific package
pnpm add zod --filter @myapp/api
pnpm add -D vitest --filter @myapp/ui

# Add an internal workspace dep
pnpm add @myapp/types --filter @myapp/api --workspace

# Interactive package selection
turbo run build --filter=$(pnpm ls -r --json \
  | python3 -c "import json,sys; \
    pkgs=json.load(sys.stdin); \
    print(','.join(p['name'] for p in pkgs if 'test' not in p.get('name','')))")
```
