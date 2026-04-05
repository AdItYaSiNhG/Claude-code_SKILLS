---
name: directory-layout
description: >
  Use this skill for project directory structure, file organisation, and layout
  decisions: scaffolding new projects, auditing and fixing existing directory structures,
  enforcing separation of concerns through folders, designing barrel file strategy,
  module aliasing, shared config package layout, and cross-app type sharing structures.
  Triggers: "directory structure", "folder structure", "project layout", "scaffold project",
  "file organisation", "where should I put", "barrel files", "path aliases",
  "shared types", "project structure", "src layout".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Directory Layout Skill

## Why this skill exists

Poor directory structure is the earliest form of architecture rot. Files in the
wrong place, flat structures that grow past 20 files, missing separation between
concerns — all of these cause navigation overhead and import coupling that
compounds over years.

This skill maps the right directory scaffold to the project type, audits
existing structures for common problems, and generates the conventions
(barrel files, aliases, shared config) that keep layouts maintainable at scale.

---

## 0. Analyse existing structure first

```bash
# Current tree (clean view)
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.py" \
  -o -name "*.go" -o -name "*.java" \) \
  | grep -v "node_modules\|.git\|dist\|build\|__pycache__\|coverage\|.next" \
  | sort | head -80

# Flat directories (>20 files = likely needs splitting)
python3 - <<'EOF'
from pathlib import Path
from collections import defaultdict

file_counts = defaultdict(int)
for f in Path(".").rglob("*"):
    if f.is_file() and "node_modules" not in str(f) and ".git" not in str(f):
        file_counts[str(f.parent)] += 1

print("Directories with >10 files (consider splitting):")
for d, count in sorted(file_counts.items(), key=lambda x: -x[1]):
    if count > 10:
        print(f"  {count:4d}  {d}")
EOF

# Deep nesting (>5 levels = over-structured)
find . -type f | grep -v "node_modules\|.git\|dist" \
  | awk -F'/' '{print NF-1, $0}' | awk '$1 > 5' | sort -rn | head -20
```

---

## 1. TypeScript / Node.js — backend API

### Feature-first (recommended for medium-large projects)
```
src/
├── features/                      # each feature owns its full stack
│   ├── users/
│   │   ├── users.controller.ts    # HTTP layer
│   │   ├── users.service.ts       # business logic
│   │   ├── users.repository.ts    # data access
│   │   ├── users.routes.ts        # route registration
│   │   ├── users.schema.ts        # Zod/Joi validation schemas
│   │   ├── users.types.ts         # types local to this feature
│   │   ├── users.test.ts          # co-located unit tests
│   │   └── index.ts               # barrel — public API of feature
│   ├── orders/
│   └── payments/
│
├── shared/                        # cross-feature utilities
│   ├── db/
│   │   ├── client.ts              # Prisma/Drizzle instance (singleton)
│   │   └── migrations/
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   ├── error.middleware.ts
│   │   └── request-id.middleware.ts
│   ├── errors/
│   │   └── index.ts               # AppError, NotFoundError, etc.
│   ├── utils/
│   │   ├── logger.ts
│   │   └── pagination.ts
│   └── types/
│       └── index.ts               # global shared types
│
├── config/
│   ├── env.ts                     # validated env vars (zod)
│   ├── database.ts
│   └── redis.ts
│
├── app.ts                         # express/fastify setup (no business logic)
└── server.ts                      # entry point (listen, signals)

tests/
├── integration/                   # integration tests (separate from unit)
├── e2e/                           # E2E tests
└── factories/                     # test data factories

docs/
└── adr/                           # architecture decision records
```

### Layer-first (simpler, fine for small projects)
```
src/
├── controllers/
├── services/
├── repositories/
├── models/                        # or schemas/ for Zod/Prisma types
├── middleware/
├── routes/
├── utils/
└── types/
```

---

## 2. React / Next.js — frontend

### Next.js App Router (recommended — 2024+)
```
src/
├── app/                           # Next.js App Router
│   ├── (auth)/                    # route group — doesn't affect URL
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx             # dashboard shell layout
│   │   ├── page.tsx               # /dashboard
│   │   └── settings/
│   │       └── page.tsx
│   ├── api/                       # API routes
│   │   └── users/
│   │       └── route.ts
│   ├── layout.tsx                 # root layout
│   └── globals.css
│
├── components/                    # shared UI components
│   ├── ui/                        # atomic/primitive (Button, Input, Modal)
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   └── index.ts               # barrel
│   ├── forms/                     # form compositions
│   └── layouts/                   # layout components
│
├── features/                      # domain-specific components + logic
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── AuthGuard.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   └── index.ts
│   ├── orders/
│   └── products/
│
├── hooks/                         # shared custom hooks
│   ├── useDebounce.ts
│   └── usePagination.ts
│
├── lib/                           # third-party config and utilities
│   ├── api/                       # API client (axios/fetch wrappers)
│   │   └── client.ts
│   ├── query/                     # React Query setup
│   │   └── queryClient.ts
│   └── auth/                      # NextAuth config
│       └── options.ts
│
├── store/                         # global state (Zustand/Redux)
│   ├── useUserStore.ts
│   └── useCartStore.ts
│
└── types/                         # global TypeScript types
    ├── api.ts                     # API response shapes
    └── index.ts
```

### Feature-sliced design (FSD — large teams)
```
src/
├── app/                           # app init, providers, global styles
├── pages/                         # routing layer only — thin wrappers
├── widgets/                       # composite UI blocks (navbar, sidebar)
├── features/                      # user interactions (search-bar, add-to-cart)
├── entities/                      # business objects (User, Order, Product)
└── shared/                        # reusable atoms (ui kit, lib, api, types)
    ├── ui/
    ├── api/
    ├── lib/
    └── types/
```

---

## 3. Python — FastAPI / Django

### FastAPI (domain-oriented)
```
app/
├── api/
│   ├── deps.py                    # FastAPI dependency injection
│   ├── router.py                  # root router
│   └── v1/
│       ├── router.py              # v1 router
│       └── endpoints/
│           ├── users.py
│           └── orders.py
│
├── core/
│   ├── config.py                  # pydantic Settings
│   ├── security.py                # JWT, password hashing
│   └── logging.py
│
├── db/
│   ├── base.py                    # SQLAlchemy declarative base
│   ├── session.py                 # engine + session factory
│   └── migrations/                # Alembic
│       └── versions/
│
├── domains/                       # feature modules
│   ├── users/
│   │   ├── models.py              # SQLAlchemy models
│   │   ├── schemas.py             # Pydantic schemas
│   │   ├── repository.py          # DB queries
│   │   ├── service.py             # business logic
│   │   └── exceptions.py
│   └── orders/
│
├── shared/
│   ├── exceptions.py
│   └── pagination.py
│
└── main.py                        # FastAPI app factory

tests/
├── conftest.py                    # pytest fixtures (test DB, client)
├── unit/
├── integration/
└── factories/
```

### Django (conventional)
```
myproject/
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
│
├── apps/
│   ├── users/
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── serializers.py         # DRF
│   │   ├── urls.py
│   │   ├── admin.py
│   │   ├── signals.py
│   │   └── tests/
│   └── orders/
│
├── shared/
│   ├── models.py                  # abstract base models (timestamps)
│   └── exceptions.py
│
└── manage.py
```

---

## 4. Go — microservice

```
myservice/
├── cmd/
│   └── server/
│       └── main.go                # entry point only (wire + listen)
│
├── internal/                      # private — cannot be imported by other modules
│   ├── domain/                    # entities, value objects, interfaces
│   │   ├── user.go
│   │   └── repository.go          # interfaces
│   ├── service/                   # use cases
│   │   └── user_service.go
│   ├── handler/                   # HTTP handlers
│   │   └── user_handler.go
│   ├── repository/                # DB implementations
│   │   └── postgres_user_repo.go
│   └── middleware/
│       ├── auth.go
│       └── logger.go
│
├── pkg/                           # public — can be imported by other modules
│   ├── logger/
│   └── errors/
│
├── migrations/
│   └── 001_create_users.sql
│
├── config/
│   └── config.go                  # config struct + loader
│
└── docker/
    └── Dockerfile
```

---

## 5. Java / Kotlin — Spring Boot

```
src/
├── main/
│   ├── java/com/company/app/
│   │   ├── Application.java       # entry point
│   │   ├── config/                # Spring config beans
│   │   ├── domain/
│   │   │   ├── model/             # entities
│   │   │   ├── repository/        # JPA repository interfaces
│   │   │   └── service/           # business logic
│   │   ├── web/
│   │   │   ├── controller/        # REST controllers
│   │   │   ├── dto/               # request/response DTOs
│   │   │   └── mapper/            # entity ↔ DTO mappers (MapStruct)
│   │   ├── infrastructure/
│   │   │   ├── persistence/       # custom queries, JPA impl
│   │   │   └── messaging/         # Kafka producers/consumers
│   │   └── shared/
│   │       ├── exception/
│   │       └── util/
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       └── db/migration/          # Flyway migrations
└── test/
    ├── unit/
    └── integration/
```

---

## 6. Barrel files (index.ts) — when and how

Barrel files aggregate exports so callers import from a path, not a file:

```typescript
// WITHOUT barrel
import { UserService } from "../../features/users/users.service";
import { UserSchema } from "../../features/users/users.schema";
import { UserController } from "../../features/users/users.controller";

// WITH barrel (features/users/index.ts)
export { UserService } from "./users.service";
export { UserSchema } from "./users.schema";
// Controller is intentionally NOT exported — it's internal to the feature

// Caller
import { UserService, UserSchema } from "../../features/users";
```

**Rules for barrel files:**
- One `index.ts` per feature/module — export only the **public API**
- Never re-export everything: `export * from "./..."` — hides what's public vs private
- Never create barrels in `node_modules`-like flat directories — it adds no value
- Deep barrel chains (index → index → index) cause circular dep issues in bundlers

```bash
# Find barrel files
find src -name "index.ts" | while read f; do
  exports=$(grep -c "^export" "$f" 2>/dev/null)
  echo "$exports exports  $f"
done | sort -rn | head -20

# Detect barrel files with re-export-all (risky pattern)
grep -rn "^export \* from" --include="index.ts" . | grep -v node_modules | head -20
```

---

## 7. Path aliases

Eliminate `../../..` relative imports with TypeScript path aliases:

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*":          ["src/*"],
      "@features/*":  ["src/features/*"],
      "@shared/*":    ["src/shared/*"],
      "@config/*":    ["src/config/*"],
      "@types/*":     ["src/types/*"]
    }
  }
}
```

```javascript
// vite.config.ts or webpack.config.js (must match tsconfig)
import { resolve } from "path";

export default {
  resolve: {
    alias: {
      "@":         resolve(__dirname, "src"),
      "@features": resolve(__dirname, "src/features"),
      "@shared":   resolve(__dirname, "src/shared"),
    },
  },
};
```

```bash
# Find deep relative imports (candidates for alias replacement)
grep -rn "from ['\"]\.\.\/\.\.\/" --include="*.ts" --include="*.tsx" \
  . | grep -v "node_modules" | head -30

# Count depth violations (3+ levels = should use alias)
grep -rn "from ['\"]" --include="*.ts" --include="*.tsx" \
  . | grep -v node_modules \
  | grep -oP "'\.\.(\/\.\.){2,}[^']*'" | sort | uniq -c | sort -rn | head -20
```

---

## 8. Environment configuration layout

```
├── .env.example           # committed — documents all required vars (no values)
├── .env.local             # gitignored — developer local overrides
├── .env.test              # committed — test environment defaults
├── .env.staging           # gitignored (or in secrets manager)
└── .env.production        # NEVER committed — use secrets manager

# Validated config loader (TypeScript)
# src/config/env.ts
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV:       z.enum(["development", "test", "production"]),
  PORT:           z.coerce.number().default(3000),
  DATABASE_URL:   z.string().url(),
  JWT_SECRET:     z.string().min(32),
  REDIS_URL:      z.string().url().optional(),
  LOG_LEVEL:      z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export const env = envSchema.parse(process.env);
// App crashes at startup if any required var is missing — fail fast
```

---

## 9. Shared config packages (monorepo)

```
packages/
└── config/
    ├── package.json           # name: "@myapp/config"
    ├── tsconfig.base.json     # base TypeScript config
    ├── eslint-config/
    │   ├── index.js           # base ESLint config
    │   ├── react.js           # extends base + React rules
    │   └── next.js
    ├── prettier/
    │   └── index.js           # shared Prettier config
    └── jest/
        ├── base.config.js     # base Jest config
        └── react.config.js    # extends base + jsdom
```

```json
// packages/config/tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

---

## 10. Layout audit report

```bash
python3 - <<'EOF'
from pathlib import Path
from collections import defaultdict
import re

issues = []

# 1. Flat directories with too many files
dir_files = defaultdict(list)
for f in Path("src").rglob("*"):
    if f.is_file() and "node_modules" not in str(f):
        dir_files[str(f.parent)].append(f.name)

for d, files in dir_files.items():
    if len(files) > 15:
        issues.append(f"FLAT  {d}/ has {len(files)} files — consider splitting by feature")

# 2. Deep relative imports (>2 levels up)
for f in Path("src").rglob("*.ts"):
    if "node_modules" in str(f): continue
    try:
        text = f.read_text(errors="replace")
        for m in re.finditer(r"from ['\"](\.\.(\/\.\.){2,}[^'\"]*)['\"]", text):
            issues.append(f"DEEP  {f}: import '{m.group(1)}'")
    except: pass

# 3. Test files mixed with source (fine for co-located, but flag deep nesting)
for f in Path("src").rglob("*.test.ts"):
    if "node_modules" in str(f): continue
    depth = len(f.parts)
    if depth > 6:
        issues.append(f"NEST  {f} (depth {depth}) — deeply nested test")

# 4. Missing index.ts in feature directories
for d in Path("src/features").iterdir() if Path("src/features").exists() else []:
    if d.is_dir() and not (d / "index.ts").exists():
        issues.append(f"BARREL  {d}/ missing index.ts barrel")

print(f"Layout issues: {len(issues)}")
for i in issues[:30]: print(f"  {i}")
EOF
```
