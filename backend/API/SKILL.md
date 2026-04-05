---
name: backend-api
description: >
  Use this skill for all backend API tasks: REST API design, GraphQL schema design,
  gRPC services, WebSocket handling, tRPC, OpenAPI spec generation, API mock servers,
  authentication/authorisation patterns, rate limiting, request validation, error
  handling, and API versioning. Covers Node.js/Express, FastAPI, Go, and Spring Boot.
  Triggers: "REST API", "GraphQL", "gRPC", "WebSocket", "tRPC", "OpenAPI", "Swagger",
  "API design", "API versioning", "auth middleware", "rate limit", "request validation",
  "API mock", "endpoint", "route", "controller".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Backend API Skill

## Why this skill exists

APIs are contracts. A badly designed API propagates bad patterns to every client
for years. This skill enforces **consistent API design — correct HTTP semantics,
strict validation, layered auth, structured errors, and generated documentation** —
across all major backend frameworks and API paradigms.

---

## 0. Audit existing API

```bash
# Detect API framework
cat package.json 2>/dev/null | python3 -c "
import json,sys; d=json.load(sys.stdin)
all_deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
for f in ['express','fastify','hono','koa','nestjs','@nestjs/core','trpc','@trpc/server']:
    if f in all_deps: print(f'{f}: {all_deps[f]}')
"

# Find all route definitions
grep -rn "\.get(\|\.post(\|\.put(\|\.patch(\|\.delete(\|router\." \
  --include="*.ts" --include="*.js" . \
  | grep -v "node_modules\|test\|spec" | head -30

# Python framework
grep -rn "@app\.\|@router\.\|APIRouter\|FastAPI" \
  --include="*.py" . | grep -v "test\|venv" | head -20

# Count endpoints
grep -rn "app\.\(get\|post\|put\|patch\|delete\)\|@router\.\|@app\." \
  --include="*.ts" --include="*.py" . | grep -v "node_modules\|test" | wc -l
```

---

## 1. REST API design

### HTTP semantics
```
GET    /users           → 200 + list    (idempotent, cacheable)
GET    /users/:id       → 200 + item | 404
POST   /users           → 201 + created item + Location header
PUT    /users/:id       → 200 + updated | 404   (full replacement)
PATCH  /users/:id       → 200 + updated | 404   (partial update)
DELETE /users/:id       → 204 no body | 404

Nested resources:
GET    /users/:id/orders          → user's orders
POST   /users/:id/orders          → create order for user
DELETE /users/:id/orders/:orderId → delete specific order

Query params:
GET /users?page=1&limit=20&sort=name:asc&filter[role]=admin
GET /products?q=laptop&category=electronics&minPrice=100
```

### Express route structure (TypeScript)
```typescript
// src/features/users/users.routes.ts
import { Router } from "express";
import { authenticate } from "@/shared/middleware/auth";
import { requireRole } from "@/shared/middleware/rbac";
import { validate } from "@/shared/middleware/validate";
import { UsersController } from "./users.controller";
import { CreateUserSchema, UpdateUserSchema, UserQuerySchema } from "./users.schema";

export function usersRouter(controller: UsersController): Router {
  const router = Router();

  router.get("/",         authenticate, validate("query", UserQuerySchema),  controller.list);
  router.get("/:id",      authenticate,                                       controller.getById);
  router.post("/",        authenticate, requireRole("admin"), validate("body", CreateUserSchema), controller.create);
  router.patch("/:id",    authenticate, validate("body", UpdateUserSchema),   controller.update);
  router.delete("/:id",   authenticate, requireRole("admin"),                 controller.delete);

  return router;
}

// src/features/users/users.controller.ts
export class UsersController {
  constructor(private readonly service: UsersService) {}

  list = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { page = 1, limit = 20, sort, filter } = req.query as UserQuery;
      const result = await this.service.list({ page: +page, limit: +limit, sort, filter });
      res.json({ data: result.items, meta: { page: +page, limit: +limit, total: result.total } });
    } catch (err) { next(err); }
  };

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.service.create(req.body);
      res.status(201).location(`/users/${user.id}`).json({ data: user });
    } catch (err) { next(err); }
  };
}
```

### Request validation (Zod)
```typescript
// src/features/users/users.schema.ts
import { z } from "zod";

export const CreateUserSchema = z.object({
  name:  z.string().min(1).max(100).trim(),
  email: z.string().email().toLowerCase(),
  role:  z.enum(["user","admin","viewer"]).default("user"),
  password: z.string().min(8).max(128).regex(/[A-Z]/).regex(/[0-9]/),
});

export const UpdateUserSchema = CreateUserSchema.partial().omit({ password: true });

export const UserQuerySchema = z.object({
  page:  z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sort:  z.string().regex(/^[a-zA-Z]+:(asc|desc)$/).optional(),
  "filter[role]": z.enum(["user","admin","viewer"]).optional(),
});

// Validation middleware
export function validate(source: "body" | "query" | "params", schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req[source]);
    if (!result.success) {
      return res.status(400).json({
        error: {
          code:    "VALIDATION_ERROR",
          message: "Invalid request data",
          fields:  result.error.flatten().fieldErrors,
        },
      });
    }
    req[source] = result.data;
    next();
  };
}
```

### Standardised error responses
```typescript
// Always return this shape for errors
{
  "error": {
    "code":    "NOT_FOUND",           // machine-readable
    "message": "User not found",      // human-readable
    "details": { "userId": "abc123" } // optional context
  }
}

// Error codes
NOT_FOUND          → 404
VALIDATION_ERROR   → 400
UNAUTHORIZED       → 401
FORBIDDEN          → 403
CONFLICT           → 409
RATE_LIMITED       → 429
INTERNAL_ERROR     → 500
```

### Pagination response shape
```typescript
// Always consistent across all list endpoints
{
  "data": [...],
  "meta": {
    "page":       1,
    "limit":      20,
    "total":      243,
    "totalPages": 13,
    "hasNext":    true,
    "hasPrev":    false
  },
  "links": {
    "self":  "/api/users?page=1&limit=20",
    "next":  "/api/users?page=2&limit=20",
    "prev":  null,
    "first": "/api/users?page=1&limit=20",
    "last":  "/api/users?page=13&limit=20"
  }
}
```

---

## 2. Authentication & authorisation

### JWT middleware
```typescript
// src/shared/middleware/auth.ts
import jwt from "jsonwebtoken";

export async function authenticate(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    return res.status(401).json({ error: { code: "UNAUTHORIZED", message: "Bearer token required" } });
  }

  const token = header.slice(7);
  try {
    const payload = jwt.verify(token, env.JWT_SECRET, { algorithms: ["RS256"] }) as JWTPayload;
    req.user = { id: payload.sub, role: payload.role, sessionId: payload.jti };
    next();
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: { code: "TOKEN_EXPIRED", message: "Token expired" } });
    }
    return res.status(401).json({ error: { code: "INVALID_TOKEN", message: "Invalid token" } });
  }
}

// RBAC middleware
export function requireRole(...roles: UserRole[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: { code: "FORBIDDEN", message: "Insufficient permissions" } });
    }
    next();
  };
}
```

### Rate limiting
```typescript
import rateLimit from "express-rate-limit";
import RedisStore from "rate-limit-redis";

// Global rate limit
export const globalRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15 min
  max: 500,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ client: redis }),
  message: { error: { code: "RATE_LIMITED", message: "Too many requests" } },
});

// Strict rate limit for auth endpoints
export const authRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  keyGenerator: (req) => req.ip + req.path,
  store: new RedisStore({ client: redis }),
});

app.use(globalRateLimit);
app.use("/api/auth", authRateLimit);
```

---

## 3. GraphQL (with Pothos / type-graphql)

```typescript
// src/graphql/schema.ts — Pothos (code-first, fully typed)
import SchemaBuilder from "@pothos/core";
import PrismaPlugin from "@pothos/plugin-prisma";

const builder = new SchemaBuilder<{ PrismaTypes: PrismaTypes }>({
  plugins: [PrismaPlugin],
  prisma: { client: db },
});

builder.prismaObject("User", {
  fields: (t) => ({
    id:        t.exposeID("id"),
    name:      t.exposeString("name"),
    email:     t.exposeString("email"),
    role:      t.exposeString("role"),
    createdAt: t.expose("createdAt", { type: "DateTime" }),
    orders:    t.relation("orders"),
  }),
});

builder.queryFields((t) => ({
  user: t.prismaField({
    type: "User",
    nullable: true,
    args: { id: t.arg.id({ required: true }) },
    resolve: (query, _, args, ctx) => {
      if (!ctx.user) throw new GraphQLError("Unauthorized", { extensions: { code: "UNAUTHORIZED" } });
      return db.user.findUnique({ ...query, where: { id: args.id } });
    },
  }),

  users: t.prismaField({
    type: ["User"],
    args: {
      first:  t.arg.int(),
      after:  t.arg.string(),
      filter: t.arg({ type: UserFilterInput }),
    },
    resolve: (query, _, args, ctx) => {
      return db.user.findMany({
        ...query,
        take: args.first ?? 20,
        cursor: args.after ? { id: args.after } : undefined,
        where: args.filter ?? {},
      });
    },
  }),
}));

builder.mutationFields((t) => ({
  createUser: t.prismaField({
    type: "User",
    args: { input: t.arg({ type: CreateUserInput, required: true }) },
    resolve: (query, _, args, ctx) => {
      if (ctx.user?.role !== "admin") throw new GraphQLError("Forbidden");
      return db.user.create({ ...query, data: args.input });
    },
  }),
}));

export const schema = builder.toSchema();
```

---

## 4. gRPC service

```protobuf
// proto/users/v1/users.proto
syntax = "proto3";
package users.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  // Server-streaming for large datasets
  rpc StreamUsers(ListUsersRequest) returns (stream User);
}

message User {
  string id         = 1;
  string name       = 2;
  string email      = 3;
  string role       = 4;
  google.protobuf.Timestamp created_at = 5;
}

message GetUserRequest    { string id = 1; }
message DeleteUserRequest { string id = 1; }
message CreateUserRequest { string name = 1; string email = 2; string role = 3; }
message UpdateUserRequest { string id = 1; google.protobuf.FieldMask update_mask = 2; string name = 3; }
message ListUsersRequest  { int32 page_size = 1; string page_token = 2; string filter = 3; }
message ListUsersResponse { repeated User users = 1; string next_page_token = 2; int32 total_size = 3; }
```

```typescript
// src/grpc/users.handler.ts
import { ServerUnaryCall, sendUnaryData, status } from "@grpc/grpc-js";

export const usersGrpcHandler = {
  async getUser(call: ServerUnaryCall<GetUserRequest, User>, callback: sendUnaryData<User>) {
    try {
      const user = await userService.findById(call.request.id);
      if (!user) return callback({ code: status.NOT_FOUND, message: "User not found" });
      callback(null, userToProto(user));
    } catch (err) {
      callback({ code: status.INTERNAL, message: "Internal error" });
    }
  },
};
```

---

## 5. tRPC (full-stack type safety)

```typescript
// src/server/trpc/router.ts
import { initTRPC, TRPCError } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.context<TRPCContext>().create();

export const publicProcedure    = t.procedure;
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session?.user) throw new TRPCError({ code: "UNAUTHORIZED" });
  return next({ ctx: { ...ctx, user: ctx.session.user } });
});

export const appRouter = t.router({
  users: t.router({
    list: protectedProcedure
      .input(z.object({ page: z.number().default(1), limit: z.number().max(100).default(20) }))
      .query(async ({ input, ctx }) => {
        const [items, total] = await Promise.all([
          db.user.findMany({ skip: (input.page-1)*input.limit, take: input.limit }),
          db.user.count(),
        ]);
        return { items, total, page: input.page, limit: input.limit };
      }),

    byId: protectedProcedure
      .input(z.string().cuid())
      .query(async ({ input }) => {
        const user = await db.user.findUnique({ where: { id: input } });
        if (!user) throw new TRPCError({ code: "NOT_FOUND" });
        return user;
      }),

    create: protectedProcedure
      .input(CreateUserSchema)
      .mutation(async ({ input, ctx }) => {
        if (ctx.user.role !== "admin") throw new TRPCError({ code: "FORBIDDEN" });
        return db.user.create({ data: input });
      }),
  }),
});

export type AppRouter = typeof appRouter;
```

---

## 6. WebSocket (real-time API)

```typescript
// src/websocket/handler.ts — using ws + typed events
import { WebSocketServer, WebSocket } from "ws";

interface TypedWebSocket extends WebSocket {
  userId?: string;
  isAlive: boolean;
}

export function setupWebSocket(server: http.Server) {
  const wss = new WebSocketServer({ server, path: "/ws" });
  const rooms = new Map<string, Set<TypedWebSocket>>();

  // Heartbeat — detect dead connections
  const heartbeat = setInterval(() => {
    wss.clients.forEach((ws: TypedWebSocket) => {
      if (!ws.isAlive) return ws.terminate();
      ws.isAlive = false;
      ws.ping();
    });
  }, 30_000);

  wss.on("connection", (ws: TypedWebSocket, req) => {
    ws.isAlive = true;
    ws.on("pong", () => { ws.isAlive = true; });

    ws.on("message", (raw) => {
      const msg = JSON.parse(raw.toString()) as WSMessage;

      switch (msg.type) {
        case "subscribe":
          if (!rooms.has(msg.room)) rooms.set(msg.room, new Set());
          rooms.get(msg.room)!.add(ws);
          ws.send(JSON.stringify({ type: "subscribed", room: msg.room }));
          break;

        case "publish":
          const subscribers = rooms.get(msg.room);
          subscribers?.forEach(client => {
            if (client !== ws && client.readyState === WebSocket.OPEN) {
              client.send(JSON.stringify({ type: "message", room: msg.room, data: msg.data }));
            }
          });
          break;
      }
    });

    ws.on("close", () => {
      rooms.forEach(clients => clients.delete(ws));
    });
  });

  wss.on("close", () => clearInterval(heartbeat));
  return wss;
}
```

---

## 7. OpenAPI spec generation

```typescript
// Fastify with @fastify/swagger — auto-generates from route schemas
import fastifySwagger from "@fastify/swagger";
import fastifySwaggerUI from "@fastify/swagger-ui";

await app.register(fastifySwagger, {
  openapi: {
    info:       { title: "My API", version: "1.0.0", description: "API description" },
    servers:    [{ url: "https://api.myapp.com", description: "Production" }],
    components: {
      securitySchemes: {
        bearerAuth: { type: "http", scheme: "bearer", bearerFormat: "JWT" },
      },
    },
    security: [{ bearerAuth: [] }],
  },
});

await app.register(fastifySwaggerUI, { routePrefix: "/docs" });

// Route with schema — auto-documented
app.get("/users/:id", {
  schema: {
    tags: ["Users"],
    summary: "Get user by ID",
    params:   z.object({ id: z.string().cuid() }),
    response: {
      200: z.object({ data: UserSchema }),
      404: ErrorSchema,
    },
    security: [{ bearerAuth: [] }],
  },
}, getUserHandler);
```

```bash
# Export OpenAPI spec
curl http://localhost:3000/docs/json 2>/dev/null | python3 -m json.tool > openapi.json

# Validate spec
npx @redocly/cli lint openapi.json 2>/dev/null

# Generate client SDK from spec
npx openapi-generator-cli generate \
  -i openapi.json -g typescript-fetch \
  -o src/generated/api-client 2>/dev/null
```

---

## 8. FastAPI (Python)

```python
# app/api/v1/endpoints/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Query
from app.core.security import get_current_user, require_role
from app.domains.users.service import UserService
from app.domains.users.schemas import UserCreate, UserUpdate, UserResponse, UserListResponse

router = APIRouter(prefix="/users", tags=["Users"])

@router.get("/", response_model=UserListResponse)
async def list_users(
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100),
    sort: str = Query("created_at:desc", regex=r"^\w+:(asc|desc)$"),
    service: UserService = Depends(),
    current_user = Depends(get_current_user),
):
    items, total = await service.list(page=page, limit=limit, sort=sort)
    return {"data": items, "meta": {"page": page, "limit": limit, "total": total}}

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: str,
    service: UserService = Depends(),
    _=Depends(get_current_user),
):
    user = await service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found")
    return {"data": user}

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    body: UserCreate,
    service: UserService = Depends(),
    _=Depends(require_role("admin")),
):
    user = await service.create(body)
    return {"data": user}
```

---

## 9. API health & diagnostics

```typescript
// Health check endpoint — used by load balancers, k8s probes
app.get("/health", async (req, res) => {
  const checks = await Promise.allSettled([
    db.$queryRaw`SELECT 1`,                     // DB liveness
    redis.ping(),                               // Cache liveness
  ]);

  const [db_ok, redis_ok] = checks.map(c => c.status === "fulfilled");
  const status = db_ok && redis_ok ? 200 : 503;

  res.status(status).json({
    status: status === 200 ? "healthy" : "degraded",
    version: process.env.APP_VERSION ?? "unknown",
    checks: { database: db_ok ? "ok" : "error", cache: redis_ok ? "ok" : "error" },
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

// Readiness probe (k8s) — stricter than liveness
app.get("/ready", async (req, res) => {
  try {
    await db.$queryRaw`SELECT 1`;
    res.status(200).json({ ready: true });
  } catch {
    res.status(503).json({ ready: false });
  }
});
```

---

## 10. API test checklist

```bash
# Test all endpoints with httpie or curl
# Auth
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test123"}' | python3 -m json.tool

# Protected endpoint
curl http://localhost:3000/api/users \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Validation error
curl -X POST http://localhost:3000/api/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"not-an-email"}' | python3 -m json.tool

# Unauthenticated request → should 401
curl http://localhost:3000/api/users | python3 -m json.tool

# Rate limit test
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:3000/api/auth/login \
    -d '{"email":"x","password":"y"}'
done
```
