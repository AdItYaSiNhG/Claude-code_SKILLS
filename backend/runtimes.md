---
name: backend-runtimes
description: Language-specific runtime patterns, toolchain, and best practices for Node.js, Python, Go, Java, Rust, Ruby, and PHP. Use for idiomatic code patterns, dependency management, error handling conventions, and runtime tuning.
---

The user is working in a specific language runtime. Apply the relevant section. When language is ambiguous, ask — don't assume.

## Node.js / TypeScript

```ts
// Error handling — always typed errors
class AppError extends Error {
  constructor(public code: string, message: string, public statusCode = 500) {
    super(message)
    this.name = 'AppError'
  }
}

// Async pattern — never unhandled promise rejections
process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled rejection', { reason })
  process.exit(1)
})

// Worker threads for CPU-bound work (never block event loop)
import { Worker } from 'worker_threads'
```

**Toolchain**: pnpm (monorepo), tsx (dev), tsup (build), Vitest (test), Biome (lint+format)

**Runtime tuning**:
```bash
node --max-old-space-size=4096 \  # match container memory limit
     --enable-source-maps \        # production stack traces
     dist/server.js
```

## Python

```python
# Type hints everywhere (Python 3.12+)
from typing import TypeAlias
OrderId: TypeAlias = str

# Dataclasses > dicts for internal data
from dataclasses import dataclass, field
@dataclass(frozen=True, slots=True)
class Order:
    id: str
    total_cents: int
    tags: list[str] = field(default_factory=list)

# Async with proper cancellation handling
import asyncio
async def fetch_orders(ids: list[str]) -> list[Order]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch_one(id)) for id in ids]
    return [t.result() for t in tasks]
```

**Toolchain**: uv (package management, replaces pip+venv), ruff (lint+format), mypy (types), pytest (test)

**Performance**: use `__slots__` on hot-path classes, `functools.cache` for pure functions, `asyncio.gather` for concurrent I/O, multiprocessing for CPU-bound work.

## Go

```go
// Error wrapping — always add context
func GetOrder(ctx context.Context, id string) (*Order, error) {
    order, err := db.QueryOrder(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("GetOrder %s: %w", id, err)
    }
    return order, nil
}

// Context propagation — always first param
func ProcessOrder(ctx context.Context, order *Order) error { ... }

// Goroutine leak prevention — always cancel/timeout
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

**Toolchain**: `go mod` (deps), `golangci-lint` (linting, configure via `.golangci.yml`), `testify` (assertions), `govulncheck` (vuln scanning)

**Build**: `CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s"` for minimal Docker images

## Java (Spring Boot 3+)

```java
// Records for DTOs (Java 17+)
public record OrderResponse(
    UUID id,
    String status,
    BigDecimal totalUsd
) {}

// Virtual threads (Java 21+) — enable in Spring Boot
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreads() {
    return handler -> handler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}

// Structured logging with Logback
private static final Logger log = LoggerFactory.getLogger(OrderService.class);
log.atInfo().addKeyValue("orderId", id).addKeyValue("status", "confirmed").log("Order confirmed");
```

**Toolchain**: Maven or Gradle (Kotlin DSL preferred), Testcontainers (integration tests), ArchUnit (architecture enforcement), GraalVM native-image for CLI tools

**JVM tuning**: `-XX:+UseG1GC -Xms512m -Xmx4g -XX:MaxGCPauseMillis=200`

## Rust

```rust
// Error handling with thiserror
#[derive(Debug, thiserror::Error)]
pub enum OrderError {
    #[error("Order {0} not found")]
    NotFound(Uuid),
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
}

// Async runtime — always Tokio
#[tokio::main]
async fn main() -> anyhow::Result<()> { ... }

// Zero-copy serialization with serde
#[derive(serde::Serialize, serde::Deserialize)]
pub struct Order { pub id: Uuid, pub status: OrderStatus }
```

**Toolchain**: cargo (build/test), clippy (`cargo clippy -- -D warnings`), rustfmt, cargo-audit (vuln), cargo-nextest (faster test runner)

## Ruby (Rails 7+)

```ruby
# Service objects for business logic (never in models or controllers)
class Orders::ConfirmService
  def initialize(order, current_user)
    @order = order
    @current_user = current_user
  end

  def call
    ActiveRecord::Base.transaction do
      @order.update!(status: :confirmed)
      OrderMailer.confirmation(@order).deliver_later
      AuditLog.record(actor: @current_user, action: 'order.confirmed', subject: @order)
    end
  end
end

# Avoid N+1 — always eager load in controllers
@orders = Order.includes(:customer, :items).where(status: :pending)
```

**Toolchain**: Bundler, RuboCop (lint), RSpec (test), Brakeman (security), rack-mini-profiler (performance)

## PHP (8.3+)

```php
// Typed properties and readonly (PHP 8.1+)
readonly class OrderDto {
    public function __construct(
        public readonly string $id,
        public readonly int $totalCents,
        public readonly \DateTimeImmutable $createdAt,
    ) {}
}

// Enums (PHP 8.1+)
enum OrderStatus: string {
    case Pending   = 'pending';
    case Confirmed = 'confirmed';
    case Shipped   = 'shipped';
}

// Fibers for async (PHP 8.1+) or use ReactPHP/Swoole for event loop
```

**Toolchain**: Composer, PHPStan level 8 (strict types), Psalm, PHPUnit, PHP-CS-Fixer

## Cross-Language Guidelines

- Always handle signals for graceful shutdown (`SIGTERM` → drain in-flight requests, close DB connections)
- Log in structured JSON (key=value pairs), not interpolated strings
- Expose `/health` (liveness) and `/ready` (readiness) endpoints in every service
- Never log secrets, PII, or credentials — use structured redaction
- All timestamps in UTC, ISO 8601 format

## Output Format

1. Show idiomatic code for the specific runtime — no generic patterns
2. Note the minimum language version required for features used
3. Include toolchain commands the user can run immediately
4. Flag any deprecated patterns if the user's code uses them
