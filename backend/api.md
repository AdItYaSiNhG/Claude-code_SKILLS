---
name: backend-api
description: API design and implementation across REST, GraphQL, gRPC, tRPC, and WebSocket. Use for contract design, versioning strategy, auth patterns, rate limiting, and API documentation generation.
---

The user is designing, implementing, or auditing an API. Apply the appropriate protocol guidance below.

## Protocol Selection Matrix

| Use case | Recommended |
|---|---|
| Public/partner API, broad client support | REST (OpenAPI 3.1) |
| Flexible querying, multiple clients | GraphQL |
| Internal service-to-service, high throughput | gRPC |
| Full-stack TypeScript, end-to-end type safety | tRPC |
| Real-time bidirectional (chat, live data) | WebSocket / SSE |
| Event streaming | Kafka / NATS (not HTTP) |

## REST API Design

### URL Structure
```
GET    /v1/orders              → list (paginated)
POST   /v1/orders              → create
GET    /v1/orders/{id}         → fetch single
PATCH  /v1/orders/{id}         → partial update (prefer PATCH over PUT)
DELETE /v1/orders/{id}         → delete
GET    /v1/orders/{id}/items   → sub-resource
POST   /v1/orders/{id}/cancel  → action verb as sub-resource (not /cancelOrder)
```

### Versioning Strategy
- URI versioning (`/v1/`, `/v2/`) for public APIs — most discoverable
- Header versioning (`API-Version: 2024-01`) for internal APIs — cleaner URLs
- Never version via query param — caching breaks

### Pagination
```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6MTAwfQ==",
    "hasNextPage": true,
    "totalCount": 4821
  }
}
```
Prefer cursor-based over offset-based for large datasets (>10k rows). Offset pagination causes duplicate/missing rows on concurrent writes.

### Error Response Standard
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "issue": "Must be a valid email address" }
    ],
    "requestId": "req_01hx..."
  }
}
```
Always return `requestId` for traceability. Map to HTTP status: 400 validation, 401 unauthenticated, 403 forbidden, 404 not found, 409 conflict, 422 unprocessable, 429 rate limited, 500 internal.

### OpenAPI Generation
Prefer code-first generation (Zod → OpenAPI, TypeSpec, or Fastify schema):
```ts
// Fastify + @fastify/swagger
const schema = {
  params: z.object({ id: z.string().uuid() }),
  response: { 200: OrderSchema, 404: ErrorSchema },
}
```

## GraphQL

### Schema Design Principles
```graphql
# ✅ Use interfaces and unions for polymorphism
interface Node { id: ID! }
union SearchResult = Product | Order | Customer

# ✅ Connections pattern for pagination (Relay spec)
type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
}

# ❌ Avoid deeply nested mutations — use flat input types
```

### Performance
- Implement DataLoader for every 1-to-many relationship (eliminates N+1)
- Set `maxDepth: 10` and `maxComplexity: 1000` to prevent query abuse
- Persisted queries for production (client sends hash, not full query)
- Apollo Studio or GraphQL Hive for schema registry and change tracking

## gRPC

```protobuf
// orders.proto
syntax = "proto3";
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc ListOrders (ListOrdersRequest) returns (stream Order);  // server streaming
  rpc CreateOrder (CreateOrderRequest) returns (Order);
}
```

- Use `google.protobuf.FieldMask` for partial updates
- Generate client SDKs via `buf generate` (not `protoc` directly)
- Health check: implement `grpc.health.v1.Health` service
- Interceptors for auth, logging, and metrics (never inline in handlers)

## tRPC

```ts
// server/router/orders.ts
export const ordersRouter = router({
  list: publicProcedure
    .input(z.object({ cursor: z.string().optional() }))
    .query(async ({ input, ctx }) => { ... }),

  create: protectedProcedure
    .input(CreateOrderSchema)
    .mutation(async ({ input, ctx }) => { ... }),
})
```

- Always use `protectedProcedure` for authenticated routes
- Compose middleware via `procedure.use(middleware)` chain
- Output validation: add `.output(Schema)` to prevent internal data leakage

## Auth/AuthZ Patterns

```
Authentication  → JWT (RS256, not HS256 for multi-service), verify at API gateway
Authorisation   → RBAC for coarse-grained, ABAC/OPA for fine-grained
API Keys        → hash before storing (SHA-256), prefix with service name (sk_live_...)
OAuth2 flows    → Authorization Code + PKCE for user-facing, Client Credentials for M2M
```

## Rate Limiting

```ts
// Token bucket — recommended for API limits
// Redis-backed with sliding window
const rateLimit = {
  windowMs: 60_000,      // 1 minute
  max: 100,              // requests per window
  keyGenerator: (req) => req.user?.id ?? req.ip,
  headers: true,         // expose X-RateLimit-* headers
}
```

Always return `Retry-After` header on 429 responses.

## Output Format

For API design tasks:
1. Show the contract first (OpenAPI snippet, proto, or tRPC router type)
2. Show the implementation with error handling
3. Note auth requirements for each endpoint
4. Flag any N+1 risk, missing pagination, or missing rate limiting
