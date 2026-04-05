---
name: code-structure
description: >
  Use this skill for all code structure and software architecture tasks: domain-driven
  design (DDD) scaffolding, clean architecture enforcement, hexagonal/ports-and-adapters,
  event-driven architecture patterns, CQRS & event sourcing, saga/outbox patterns,
  monolith-to-microservice refactoring, module boundary detection, dependency graph
  visualisation, API versioning strategy, feature flag architecture, and plugin system design.
  Triggers: "architecture", "clean architecture", "DDD", "domain driven", "hexagonal",
  "ports and adapters", "CQRS", "event sourcing", "saga", "microservice", "module
  boundaries", "dependency graph", "API versioning", "plugin system", "feature flags".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Code Structure & Architecture Skill

## Why this skill exists

Architecture decisions made casually compound into structural debt. This skill
gives Claude a **pattern library and analysis toolset** for understanding existing
structure, identifying boundary violations, and scaffolding recognised enterprise
architecture patterns correctly — with working directory layouts, interface
contracts, and dependency rules.

Always analyse the existing codebase before proposing a pattern. Fit the
pattern to the domain, not the domain to the pattern.

---

## 0. Existing structure analysis

Before proposing any pattern, map what exists:

```bash
# High-level directory tree (ignore noise)
find . -type d \
  | grep -v "node_modules\|.git\|dist\|build\|__pycache__\|.next\|coverage" \
  | sort | head -60

# Detect current architecture style
python3 - <<'EOF'
from pathlib import Path

indicators = {
    "MVC / layered":         ["controllers", "models", "views", "services", "repositories"],
    "Clean / Hexagonal":     ["domain", "application", "infrastructure", "adapters", "ports"],
    "Feature-sliced":        ["features", "entities", "widgets", "pages", "shared"],
    "DDD":                   ["aggregates", "valueobjects", "domainevents", "bounded-context"],
    "Microservices":         ["services", "gateway", "api-gateway", "service-mesh"],
    "Monolith modular":      ["modules", "components", "lib", "pkg"],
}

dirs = {d.name.lower() for d in Path(".").rglob("*") if d.is_dir()
        and "node_modules" not in str(d) and ".git" not in str(d)}

print("Architecture signals detected:")
for pattern, keywords in indicators.items():
    matches = [k for k in keywords if k in dirs]
    if matches:
        print(f"  {pattern}: {matches}")
EOF

# Circular dependency check (Node/TS)
npx madge --circular --extensions ts,js src/ 2>/dev/null \
  || echo "(madge not installed — run: npm i -D madge)"

# Module coupling map
npx madge --json src/ 2>/dev/null \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
# files with most dependents (high coupling = architectural risk)
dependents = {}
for f, deps in data.items():
    for d in deps:
        dependents[d] = dependents.get(d, 0) + 1
top = sorted(dependents.items(), key=lambda x: -x[1])[:15]
print('Most imported files (high coupling):')
for f, c in top: print(f'  {c:3d}  {f}')
" 2>/dev/null
```

---

## 1. Clean Architecture

**When to use:** Services with complex business logic that must be
testable independently of frameworks, databases, and delivery mechanisms.

### Layer rules (dependency rule — always points inward)
```
Frameworks & Drivers  →  Interface Adapters  →  Application  →  Domain
(Express, Prisma, pg)     (Controllers, Repos)   (Use Cases)    (Entities)

Domain knows nothing about any outer layer.
Application knows Domain only.
Infrastructure knows everything — it wires it all together.
```

### Directory scaffold
```
src/
├── domain/                        # innermost — zero dependencies
│   ├── entities/
│   │   ├── User.ts                # pure class, no frameworks
│   │   └── Order.ts
│   ├── value-objects/
│   │   ├── Email.ts
│   │   └── Money.ts
│   ├── repositories/              # interfaces only — no implementation
│   │   └── IUserRepository.ts
│   ├── services/                  # domain services (stateless logic)
│   │   └── PricingService.ts
│   └── errors/
│       └── DomainError.ts
│
├── application/                   # use cases — depends on domain only
│   ├── use-cases/
│   │   ├── CreateUser/
│   │   │   ├── CreateUserUseCase.ts
│   │   │   ├── CreateUserDTO.ts
│   │   │   └── CreateUserUseCase.test.ts
│   │   └── GetUserById/
│   │       ├── GetUserByIdUseCase.ts
│   │       └── GetUserByIdDTO.ts
│   └── ports/                     # output ports (interfaces for infra)
│       ├── IEmailService.ts
│       └── IPaymentGateway.ts
│
├── infrastructure/                # depends on everything — wires the app
│   ├── database/
│   │   ├── prisma/
│   │   │   └── UserRepository.ts  # implements IUserRepository
│   │   └── migrations/
│   ├── http/
│   │   ├── express/
│   │   │   ├── app.ts
│   │   │   └── routes/
│   │   └── controllers/
│   │       └── UserController.ts
│   ├── services/
│   │   ├── SendGridEmailService.ts # implements IEmailService
│   │   └── StripePaymentGateway.ts
│   └── container.ts               # dependency injection wiring
│
└── main.ts                        # composition root
```

### Entity example
```typescript
// src/domain/entities/User.ts
export class User {
  private constructor(
    public readonly id: UserId,
    private _email: Email,
    private _name: string,
    private _role: UserRole,
    public readonly createdAt: Date,
  ) {}

  static create(props: { email: string; name: string }): User {
    if (!props.name.trim()) throw new DomainError("Name cannot be blank");
    return new User(
      UserId.generate(),
      Email.create(props.email),   // value object validates format
      props.name.trim(),
      UserRole.MEMBER,
      new Date(),
    );
  }

  promote(): void {
    if (this._role === UserRole.ADMIN) throw new DomainError("Already admin");
    this._role = UserRole.ADMIN;
  }

  get email(): Email { return this._email; }
  get role(): UserRole { return this._role; }
}
```

### Use case example
```typescript
// src/application/use-cases/CreateUser/CreateUserUseCase.ts
export class CreateUserUseCase {
  constructor(
    private readonly userRepo: IUserRepository,  // interface, not impl
    private readonly emailSvc: IEmailService,
  ) {}

  async execute(dto: CreateUserDTO): Promise<CreateUserResult> {
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new ConflictError("Email already registered");

    const user = User.create({ email: dto.email, name: dto.name });
    await this.userRepo.save(user);
    await this.emailSvc.sendWelcome(user.email.value, user.name);

    return { userId: user.id.value };
  }
}
```

### Enforcement check
```bash
# Verify domain has no imports from infrastructure
grep -rn "from.*infrastructure\|require.*infrastructure" src/domain/ 2>/dev/null \
  && echo "VIOLATION: domain imports infrastructure" || echo "OK: domain is clean"

# Verify application has no framework imports
grep -rn "express\|fastify\|prisma\|mongoose\|typeorm" src/application/ 2>/dev/null \
  && echo "VIOLATION: application imports framework" || echo "OK: application is clean"
```

---

## 2. Hexagonal Architecture (Ports & Adapters)

**When to use:** When you need to swap adapters (e.g., replace Postgres with
DynamoDB, replace HTTP with CLI or queue consumer) without touching business logic.

```
          ┌─────────────────────────────────────┐
  HTTP ──►│  Driving Adapter (Controller)        │
  CLI  ──►│         ↓                            │
  Queue──►│    [Input Port / Use Case]           │
          │         ↓                            │
          │    Application Hexagon               │
          │         ↓                            │
          │    [Output Port / Interface]          │◄── DB Adapter
          │         ↓                            │◄── Email Adapter
          └─────────────────────────────────────┘◄── Queue Adapter
```

### Directory scaffold
```
src/
├── core/                          # the hexagon — framework-free
│   ├── domain/                    # entities, value objects, domain events
│   ├── ports/
│   │   ├── inbound/               # driving ports (use case interfaces)
│   │   │   └── ICreateOrderPort.ts
│   │   └── outbound/              # driven ports (repo/service interfaces)
│   │       ├── IOrderRepository.ts
│   │       └── INotificationService.ts
│   └── use-cases/
│       └── CreateOrderUseCase.ts
│
└── adapters/
    ├── inbound/                   # driving adapters — call the hexagon
    │   ├── http/
    │   │   └── OrderController.ts
    │   ├── cli/
    │   │   └── ImportOrdersCommand.ts
    │   └── queue/
    │       └── OrderCreatedConsumer.ts
    └── outbound/                  # driven adapters — called by hexagon
        ├── postgres/
        │   └── PostgresOrderRepository.ts
        ├── ses/
        │   └── SESNotificationService.ts
        └── stub/                  # test doubles live here
            └── InMemoryOrderRepository.ts
```

---

## 3. Domain-Driven Design (DDD)

**When to use:** Complex domains with many business rules, multiple teams,
long-lived products.

### Bounded context mapping
```bash
# Identify potential bounded contexts from the codebase
python3 - <<'EOF'
from pathlib import Path
import re

# Large top-level groupings = likely bounded contexts
for d in sorted(Path("src").iterdir()):
    if not d.is_dir(): continue
    files = list(d.rglob("*.ts"))
    exports = 0
    for f in files:
        try: exports += f.read_text(errors="replace").count("export ")
        except: pass
    print(f"{len(files):4d} files  {exports:4d} exports  {d.name}")
EOF
```

### Scaffold: one bounded context
```
src/
└── contexts/
    ├── ordering/                  # Ordering bounded context
    │   ├── domain/
    │   │   ├── aggregates/
    │   │   │   └── Order.ts       # aggregate root
    │   │   ├── entities/
    │   │   │   └── OrderItem.ts   # child entity
    │   │   ├── value-objects/
    │   │   │   ├── OrderId.ts
    │   │   │   ├── Money.ts
    │   │   │   └── OrderStatus.ts
    │   │   ├── events/            # domain events
    │   │   │   ├── OrderPlaced.ts
    │   │   │   └── OrderCancelled.ts
    │   │   └── repositories/
    │   │       └── IOrderRepository.ts
    │   ├── application/
    │   │   ├── commands/
    │   │   │   └── PlaceOrderCommand.ts
    │   │   ├── queries/
    │   │   │   └── GetOrderQuery.ts
    │   │   └── handlers/
    │   │       ├── PlaceOrderHandler.ts
    │   │       └── GetOrderHandler.ts
    │   └── infrastructure/
    │       └── persistence/
    │           └── PrismaOrderRepository.ts
    │
    └── inventory/                 # Inventory bounded context
        └── ...
```

### Aggregate root pattern
```typescript
// src/contexts/ordering/domain/aggregates/Order.ts
export class Order extends AggregateRoot {
  private _items: OrderItem[] = [];
  private _status: OrderStatus = OrderStatus.DRAFT;

  private constructor(
    public readonly id: OrderId,
    public readonly customerId: CustomerId,
  ) { super(); }

  static place(customerId: CustomerId, items: CreateItemProps[]): Order {
    if (!items.length) throw new DomainError("Order must have at least one item");
    const order = new Order(OrderId.generate(), customerId);
    items.forEach(i => order.addItem(i));
    order._status = OrderStatus.PLACED;
    // Record domain event — consumed by other contexts via event bus
    order.addDomainEvent(new OrderPlaced(order.id, order.customerId, order.total));
    return order;
  }

  addItem(props: CreateItemProps): void {
    if (this._status !== OrderStatus.DRAFT) throw new DomainError("Cannot modify placed order");
    this._items.push(OrderItem.create(props));
  }

  get total(): Money {
    return this._items.reduce((sum, i) => sum.add(i.subtotal), Money.zero("USD"));
  }
}

// AggregateRoot base
export abstract class AggregateRoot {
  private _events: DomainEvent[] = [];
  protected addDomainEvent(event: DomainEvent): void { this._events.push(event); }
  pullDomainEvents(): DomainEvent[] { const e = [...this._events]; this._events = []; return e; }
}
```

---

## 4. CQRS & Event Sourcing

**When to use:** High-read workloads needing separate read/write models;
audit requirements; complex state reconstruction needs.

### CQRS directory scaffold
```
src/
├── commands/                      # write side
│   ├── handlers/
│   │   └── CreateOrderHandler.ts
│   ├── CreateOrderCommand.ts
│   └── CommandBus.ts
│
├── queries/                       # read side (separate model/DB)
│   ├── handlers/
│   │   └── GetOrderSummaryHandler.ts
│   ├── GetOrderSummaryQuery.ts
│   ├── read-models/
│   │   └── OrderSummaryReadModel.ts
│   └── QueryBus.ts
│
└── projections/                   # event → read model updates
    └── OrderProjection.ts
```

### Event sourcing store interface
```typescript
// src/event-store/IEventStore.ts
export interface StoredEvent {
  id: string;
  aggregateId: string;
  aggregateType: string;
  eventType: string;
  payload: unknown;
  metadata: { causedBy?: string; correlationId?: string };
  version: number;
  occurredAt: Date;
}

export interface IEventStore {
  append(aggregateId: string, events: DomainEvent[], expectedVersion: number): Promise<void>;
  load(aggregateId: string, fromVersion?: number): Promise<StoredEvent[]>;
  loadByType(aggregateType: string, afterPosition?: number): AsyncIterable<StoredEvent>;
}
```

---

## 5. Event-Driven Architecture

### Patterns available

**Simple event bus (in-process):**
```typescript
// src/shared/EventBus.ts
type Handler<T> = (event: T) => Promise<void>;

export class EventBus {
  private handlers = new Map<string, Handler<unknown>[]>();

  subscribe<T>(eventType: string, handler: Handler<T>): void {
    const existing = this.handlers.get(eventType) ?? [];
    this.handlers.set(eventType, [...existing, handler as Handler<unknown>]);
  }

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.constructor.name) ?? [];
    await Promise.all(handlers.map(h => h(event)));
  }
}
```

**Outbox pattern (guaranteed delivery across DB + message broker):**
```
src/
└── shared/
    └── outbox/
        ├── OutboxMessage.ts       # entity stored alongside domain data
        ├── IOutboxRepository.ts
        ├── OutboxPublisher.ts     # polling job that reads & publishes
        └── OutboxRelay.ts         # marks messages as published
```

```typescript
// Outbox message stored in same DB transaction as domain change
// Prevents dual-write problem (DB save ✓ but broker publish ✗)

// In your repository / unit of work:
await db.transaction(async (tx) => {
  await tx.orders.save(order);
  // Events go to outbox in same transaction — atomically
  for (const event of order.pullDomainEvents()) {
    await tx.outbox.insert({
      id: uuid(),
      eventType: event.constructor.name,
      payload: JSON.stringify(event),
      createdAt: new Date(),
      publishedAt: null,
    });
  }
});
```

---

## 6. Saga Pattern

**When to use:** Distributed transactions spanning multiple services/databases
where you cannot use a 2-phase commit.

### Choreography saga (event-driven, no central orchestrator)
```
OrderService     InventoryService     PaymentService
    │                   │                   │
    ├─ OrderPlaced ─────►                   │
    │                   ├─ StockReserved ───►
    │                   │                   ├─ PaymentCaptured
    │◄──────────────────┼───────────────────┤
    ├─ OrderConfirmed   │                   │
    │                   │                   │
   (if PaymentFailed)   │                   │
    │◄──────────────────┼── StockReleased ◄─┤
    ├─ OrderCancelled   │                   │
```

### Orchestration saga (central saga orchestrator)
```typescript
// src/sagas/PlaceOrderSaga.ts
export class PlaceOrderSaga {
  private state: SagaState = "STARTED";

  async execute(command: PlaceOrderCommand): Promise<void> {
    try {
      const orderId = await this.orderSvc.createOrder(command);
      this.state = "ORDER_CREATED";

      await this.inventorySvc.reserveStock(orderId, command.items);
      this.state = "STOCK_RESERVED";

      await this.paymentSvc.capturePayment(orderId, command.paymentMethod);
      this.state = "PAYMENT_CAPTURED";

      await this.orderSvc.confirmOrder(orderId);
      this.state = "COMPLETED";
    } catch (err) {
      await this.compensate(command.orderId, err);
    }
  }

  // Compensation — run in reverse order
  private async compensate(orderId: string, reason: Error): Promise<void> {
    if (this.state === "PAYMENT_CAPTURED")
      await this.paymentSvc.refundPayment(orderId);
    if (this.state === "STOCK_RESERVED" || this.state === "PAYMENT_CAPTURED")
      await this.inventorySvc.releaseStock(orderId);
    if (this.state !== "STARTED")
      await this.orderSvc.cancelOrder(orderId, reason.message);
    this.state = "COMPENSATED";
  }
}
```

---

## 7. Monolith → Microservice refactoring

**When to do it:** When a specific bounded context has independent scaling
needs, different deployment cadence, or different team ownership.

### Step-by-step strangler fig
```bash
# Step 1 — identify seams (high-value extraction candidates)
python3 - <<'EOF'
from pathlib import Path
import re
from collections import defaultdict

# Find modules with many inbound but few outbound dependencies
# These are good extraction candidates (already somewhat isolated)
deps = defaultdict(set)
for f in Path("src").rglob("*.ts"):
    if "node_modules" in str(f): continue
    text = f.read_text(errors="replace")
    for imp in re.findall(r"from ['\"](\.[^'\"]+)['\"]", text):
        target = (f.parent / imp).resolve()
        deps[str(f)].add(str(target))

# Count inbound refs per module directory
inbound = defaultdict(int)
for src, targets in deps.items():
    for t in targets:
        inbound[t] += 1

print("Modules sorted by inbound deps (high = core, hard to extract first):")
for mod, count in sorted(inbound.items(), key=lambda x: -x[1])[:20]:
    print(f"  {count:3d}  {mod}")
EOF
```

### Extraction checklist per service
```
Phase 1 — Isolation (in monolith)
  [ ] Identify the bounded context boundary (no cross-BC DB joins)
  [ ] Extract into a self-contained module (own DB schema / tables)
  [ ] Add anti-corruption layer (translate between contexts)
  [ ] All inter-BC calls go through an interface (not direct import)

Phase 2 — Parallel run
  [ ] Deploy extracted service alongside monolith
  [ ] Route traffic via feature flag (0% → 1% → 10% → 100%)
  [ ] Compare responses — log discrepancies
  [ ] Monolith calls new service via HTTP/gRPC (not direct import)

Phase 3 — Cutover
  [ ] 100% traffic to new service
  [ ] Remove dead code from monolith
  [ ] Decommission shared DB tables (after data migration verified)
```

---

## 8. API versioning strategy

```typescript
// Strategy 1 — URI versioning (simplest, most explicit)
// GET /api/v1/users      (old shape)
// GET /api/v2/users      (new shape)

// Express routing
const v1 = express.Router();
const v2 = express.Router();
app.use("/api/v1", v1);
app.use("/api/v2", v2);

// Strategy 2 — Header versioning (cleaner URLs)
// Accept: application/vnd.myapp.v2+json

app.use("/api/users", (req, res, next) => {
  const version = req.headers["api-version"] ?? "1";
  req.apiVersion = parseInt(version, 10);
  next();
});

// Strategy 3 — Query param (easiest for clients to test)
// GET /api/users?version=2
```

**Versioning rules:**
- Never break a published contract — add fields, never remove
- Deprecation window: minimum 6 months before removing a version
- Maintain N and N-1 versions simultaneously
- Document sunset dates in response headers: `Sunset: Sat, 01 Jan 2026 00:00:00 GMT`

---

## 9. Plugin system design

```typescript
// src/plugins/IPlugin.ts
export interface IPlugin {
  name: string;
  version: string;
  register(app: Application): Promise<void>;
  teardown?(): Promise<void>;
}

// src/plugins/PluginRegistry.ts
export class PluginRegistry {
  private plugins = new Map<string, IPlugin>();

  async register(plugin: IPlugin): Promise<void> {
    if (this.plugins.has(plugin.name))
      throw new Error(`Plugin ${plugin.name} already registered`);
    await plugin.register(this.app);
    this.plugins.set(plugin.name, plugin);
    this.logger.info({ plugin: plugin.name }, "Plugin registered");
  }

  async teardownAll(): Promise<void> {
    for (const plugin of this.plugins.values()) {
      await plugin.teardown?.();
    }
  }
}

// Example plugin
// src/plugins/stripe/StripePlugin.ts
export const StripePlugin: IPlugin = {
  name: "stripe",
  version: "1.0.0",
  async register(app) {
    app.container.bind(IPaymentGateway, new StripeGateway(app.config.STRIPE_KEY));
    app.router.post("/webhooks/stripe", StripeWebhookHandler);
  },
};
```

---

## 10. Architecture decision records (ADR)

Create an ADR for every significant architecture decision:

```bash
# ADR directory
mkdir -p docs/adr

# Template generator
cat > docs/adr/template.md << 'TEMPLATE'
# ADR-NNNN: [Short Title]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
**Deciders:** [names]

## Context
[What is the issue we're seeing that motivates this decision?]

## Decision
[What is the change that we're proposing / have agreed to implement?]

## Consequences
**Positive:**
- ...

**Negative / trade-offs:**
- ...

**Risks:**
- ...

## Alternatives considered
| Alternative | Reason rejected |
|-------------|----------------|
| Option A | ... |
| Option B | ... |
TEMPLATE

# List existing ADRs
ls docs/adr/*.md 2>/dev/null | sort
```

---

## Architecture health checks

```bash
# 1. Circular dependencies
npx madge --circular src/ 2>/dev/null | head -20

# 2. Layer violations (domain importing infra)
grep -rn "from.*infrastructure\|from.*adapters" src/domain/ src/application/ 2>/dev/null \
  | grep -v "test\|spec" | head -20

# 3. God modules (too many responsibilities)
find src -name "*.ts" | xargs wc -l 2>/dev/null \
  | awk '$1 > 400 {print $1, $2}' | sort -rn | head -10

# 4. Anemic domain model (entities with no methods — only getters/setters)
python3 - <<'EOF'
from pathlib import Path
import re

for f in Path("src/domain").rglob("*.ts"):
    text = f.read_text(errors="replace")
    methods = len(re.findall(r"(async\s+)?\w+\([^)]*\)\s*[:{]", text))
    getters = len(re.findall(r"get \w+\(\)", text))
    if methods > 0 and getters / max(methods, 1) > 0.8:
        print(f"Possibly anemic: {f} ({methods} methods, {getters} getters)")
EOF
```
