---
name: language-specific
description: >
  Use this skill for language-specific backend patterns and idioms: Node.js/Express,
  Python/FastAPI/Django, Go microservices, Java Spring Boot, Rust (Axum), .NET/C#,
  and Ruby on Rails. Covers language-specific error handling, concurrency, dependency
  injection, ORM patterns, testing idioms, and production-readiness configuration.
  Triggers: "Node.js pattern", "Python backend", "Go service", "Spring Boot",
  "Rust web", "C# .NET", "Rails", language name + "best practice" or "pattern"
  or "idiom" or "production", "express middleware", "fastapi dependency".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Language-Specific Backend Skill

## Why this skill exists

Each language has idiomatic patterns that prevent entire classes of bugs.
Node.js has the unhandled rejection problem. Go has the nil pointer and goroutine
leak problem. Python has the async/sync mixing problem. This skill documents the
**must-know patterns, anti-patterns, and production checklist** for each major
backend language.

---

## 0. Language detection

```bash
# Detect which language is in use
ls package.json requirements.txt go.mod Cargo.toml pom.xml build.gradle Gemfile 2>/dev/null \
  | while read f; do
    case "$f" in
      package.json)    echo "Node.js / TypeScript" ;;
      requirements.txt) echo "Python" ;;
      go.mod)          echo "Go" ;;
      Cargo.toml)      echo "Rust" ;;
      pom.xml)         echo "Java (Maven)" ;;
      build.gradle)    echo "Java/Kotlin (Gradle)" ;;
      Gemfile)         echo "Ruby" ;;
    esac
  done
```

---

## 1. Node.js / TypeScript

### Production app setup (Fastify — recommended over Express for new projects)
```typescript
// src/app.ts
import Fastify from "fastify";
import { TypeBoxTypeProvider } from "@fastify/type-provider-typebox";

export async function buildApp() {
  const app = Fastify({
    logger: {
      level: process.env.LOG_LEVEL ?? "info",
      ...(process.env.NODE_ENV === "development"
        ? { transport: { target: "pino-pretty" } }
        : {}),
    },
    requestIdHeader:    "x-request-id",
    requestIdLogLabel:  "requestId",
    trustProxy:         true,       // behind load balancer
    disableRequestLogging: false,
  }).withTypeProvider<TypeBoxTypeProvider>();

  // Plugins
  await app.register(import("@fastify/helmet"));
  await app.register(import("@fastify/cors"), { origin: process.env.ALLOWED_ORIGINS?.split(",") });
  await app.register(import("@fastify/compress"));
  await app.register(import("@fastify/rate-limit"), { max: 500, timeWindow: "15 minutes" });

  // Routes
  await app.register(import("./features/users/users.routes"), { prefix: "/api/v1/users" });

  // Global error handler
  app.setErrorHandler((err, req, reply) => {
    if (err instanceof AppError) {
      return reply.status(err.statusCode).send({ error: { code: err.code, message: err.message } });
    }
    if (err.validation) {  // Fastify validation error
      return reply.status(400).send({ error: { code: "VALIDATION_ERROR", message: "Invalid request", details: err.validation } });
    }
    req.log.error({ err }, "Unhandled error");
    reply.status(500).send({ error: { code: "INTERNAL_ERROR", message: "Something went wrong" } });
  });

  return app;
}
```

### Unhandled rejection / exception handling
```typescript
// src/server.ts
import { buildApp } from "./app";

async function main() {
  const app = await buildApp();
  await app.listen({ port: +process.env.PORT! || 3000, host: "0.0.0.0" });

  // Graceful shutdown
  const shutdown = async (signal: string) => {
    app.log.info({ signal }, "Shutting down…");
    await app.close();
    process.exit(0);
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT",  () => shutdown("SIGINT"));

  // Never swallow unhandled rejections — let the process crash + restart
  process.on("unhandledRejection", (reason) => {
    app.log.error({ reason }, "Unhandled rejection");
    process.exit(1);
  });
  process.on("uncaughtException", (err) => {
    app.log.error({ err }, "Uncaught exception");
    process.exit(1);
  });
}

main();
```

### Async patterns
```typescript
// WRONG — fire-and-forget without error handling
userCreated.emit("user.created", user);  // if listener throws, it's silent

// CORRECT — await async work in the request cycle
await eventBus.publish("user.created", user);

// WRONG — sequential awaits when parallel is fine
const user    = await userRepo.findById(id);
const orders  = await orderRepo.findByUserId(id);
const profile = await profileRepo.findByUserId(id);

// CORRECT — parallel with Promise.all
const [user, orders, profile] = await Promise.all([
  userRepo.findById(id),
  orderRepo.findByUserId(id),
  profileRepo.findByUserId(id),
]);

// CORRECT — parallel with individual error handling
const results = await Promise.allSettled([
  userRepo.findById(id),
  orderRepo.findByUserId(id),
]);
```

---

## 2. Python — FastAPI / async

### Production FastAPI setup
```python
# app/main.py
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware

logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await db.connect()
    logger.info("Database connected")
    yield
    # Shutdown
    await db.disconnect()
    logger.info("Database disconnected")

def create_app() -> FastAPI:
    app = FastAPI(
        title="My API",
        version="1.0.0",
        lifespan=lifespan,
        # Disable docs in production
        docs_url=None if settings.ENV == "production" else "/docs",
        redoc_url=None if settings.ENV == "production" else "/redoc",
    )

    app.add_middleware(GZipMiddleware, minimum_size=1000)
    app.add_middleware(CORSMiddleware,
        allow_origins=settings.ALLOWED_ORIGINS,
        allow_methods=["*"],
        allow_headers=["Authorization", "Content-Type"],
    )

    # Global exception handler
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError):
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": {"code": exc.code, "message": exc.message}},
        )

    @app.exception_handler(Exception)
    async def unhandled_handler(request: Request, exc: Exception):
        logger.error("Unhandled error", exc_info=exc, extra={"path": request.url.path})
        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={"error": {"code": "INTERNAL_ERROR", "message": "Something went wrong"}},
        )

    app.include_router(users_router, prefix="/api/v1/users", tags=["Users"])
    return app

app = create_app()
```

### Dependency injection pattern
```python
# app/core/dependencies.py
from functools import lru_cache
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer

security = HTTPBearer()

@lru_cache
def get_settings() -> Settings:
    return Settings()

async def get_db(settings: Settings = Depends(get_settings)) -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

async def get_current_user(
    token: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    try:
        payload = jwt.decode(token.credentials, settings.JWT_SECRET, algorithms=["RS256"])
        user = await db.get(User, payload["sub"])
        if not user: raise HTTPException(status_code=401, detail="User not found")
        return user
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### Async pitfalls
```python
# WRONG — mixing sync and async DB calls in async route
@router.get("/users")
async def list_users(db = Depends(get_db)):
    users = db.query(User).all()   # SQLAlchemy SYNC in async context — blocks event loop!
    return users

# CORRECT — use async ORM
@router.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).order_by(User.created_at.desc()))
    return result.scalars().all()

# WRONG — CPU-bound work in async route
@router.post("/compress")
async def compress_image(file: UploadFile):
    data = await file.read()
    compressed = compress(data)   # CPU-bound — blocks event loop
    return compressed

# CORRECT — offload to thread pool
import asyncio
@router.post("/compress")
async def compress_image(file: UploadFile):
    data = await file.read()
    compressed = await asyncio.get_event_loop().run_in_executor(None, compress, data)
    return compressed
```

---

## 3. Go — microservice patterns

### Project layout and idiomatic patterns
```go
// cmd/server/main.go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
    slog.SetDefault(logger)

    cfg := config.Load()
    db, err := database.Connect(cfg.DatabaseURL)
    if err != nil { logger.Error("DB connect failed", "err", err); os.Exit(1) }

    srv := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      routes.New(db, logger),
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    go func() {
        logger.Info("Server starting", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("Server error", "err", err)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        logger.Error("Shutdown error", "err", err)
    }
    logger.Info("Server stopped")
}
```

### Error handling (Go idiom)
```go
// internal/errors/errors.go — typed errors with HTTP codes
package errors

import "fmt"

type AppError struct {
    Code       string
    Message    string
    StatusCode int
    Err        error
}

func (e *AppError) Error() string { return fmt.Sprintf("%s: %s", e.Code, e.Message) }
func (e *AppError) Unwrap() error { return e.Err }

func NotFound(resource, id string) *AppError {
    return &AppError{Code: "NOT_FOUND", Message: fmt.Sprintf("%s not found: %s", resource, id), StatusCode: 404}
}

func Validation(msg string) *AppError {
    return &AppError{Code: "VALIDATION_ERROR", Message: msg, StatusCode: 400}
}

// Service usage
func (s *UserService) GetByID(ctx context.Context, id string) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil { return nil, fmt.Errorf("GetByID: %w", err) }
    if user == nil { return nil, errors.NotFound("user", id) }
    return user, nil
}
```

### Goroutine leak prevention
```go
// WRONG — goroutine that never exits
go func() {
    for msg := range ch {  // if ch is never closed, goroutine leaks
        process(msg)
    }
}()

// CORRECT — context-cancellable goroutine
func (w *Worker) Start(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return   // exits when context cancelled
        case msg := <-w.ch:
            w.process(msg)
        }
    }
}
```

---

## 4. Java — Spring Boot

### Production-ready Spring Boot
```java
// src/main/java/com/myapp/Application.java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 5000
      idle-timeout: 300000
  jpa:
    hibernate.ddl-auto: validate   # NEVER create/update in production
    open-in-view: false            # avoid N+1 via lazy loading in view layer
  cache:
    type: redis

server:
  port: ${PORT:8080}
  shutdown: graceful   # waits for in-flight requests on SIGTERM
  tomcat:
    threads.max: 200
    connection-timeout: 5s

management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  endpoint.health.show-details: when-authorized
```

```java
// Controller
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
public class UserController {

    private final UserService userService;

    @GetMapping
    public ResponseEntity<PageResponse<UserResponse>> list(
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
            @AuthenticationPrincipal UserDetails principal) {
        var result = userService.list(PageRequest.of(page, size));
        return ResponseEntity.ok(PageResponse.of(result));
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        var user = userService.create(request);
        return ResponseEntity.created(URI.create("/api/v1/users/" + user.getId()))
            .body(UserResponse.from(user));
    }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity.status(404).body(ErrorResponse.of("NOT_FOUND", ex.getMessage()));
    }
}
```

---

## 5. Rust — Axum

```rust
// src/main.rs
use axum::{Router, middleware};
use tower_http::{cors::CorsLayer, compression::CompressionLayer, trace::TraceLayer};
use std::net::SocketAddr;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt().with_env_filter("info").json().init();

    let db = Database::connect(&std::env::var("DATABASE_URL")?).await?;
    let state = AppState { db };

    let app = Router::new()
        .nest("/api/v1/users", users::router())
        .layer(middleware::from_fn(auth_middleware))
        .layer(CompressionLayer::new())
        .layer(CorsLayer::permissive())
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let addr: SocketAddr = format!("0.0.0.0:{}", std::env::var("PORT").unwrap_or("3000".into())).parse()?;
    tracing::info!("Listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(addr).await?;
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    Ok(())
}

// Handler with typed errors
async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
    Extension(current_user): Extension<CurrentUser>,
) -> Result<Json<UserResponse>, AppError> {
    let user = state.db.find_user_by_id(&id).await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound(format!("User {} not found", id)))?;
    Ok(Json(UserResponse::from(user)))
}

// Typed error → HTTP response mapping
enum AppError {
    NotFound(String),
    Unauthorized,
    Database(sqlx::Error),
    Validation(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code, message) = match self {
            Self::NotFound(m)    => (StatusCode::NOT_FOUND, "NOT_FOUND", m),
            Self::Unauthorized   => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED", "Unauthorized".into()),
            Self::Database(e)    => {
                tracing::error!(err = %e, "Database error");
                (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", "Something went wrong".into())
            },
            Self::Validation(m)  => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR", m),
        };
        (status, Json(json!({ "error": { "code": code, "message": message } }))).into_response()
    }
}
```

---

## 6. C# / .NET

```csharp
// Program.cs — .NET 8 minimal API
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
        .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));

builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts => {
        opts.Authority  = builder.Configuration["Auth:Authority"];
        opts.Audience   = builder.Configuration["Auth:Audience"];
    });

builder.Services.AddResponseCompression();
builder.Services.AddHealthChecks().AddDbContextCheck<AppDbContext>();

var app = builder.Build();
app.UseResponseCompression();
app.UseAuthentication();
app.UseAuthorization();

// Minimal API endpoints
app.MapGet("/api/users/{id}", async (string id, IUserService svc) => {
    var user = await svc.GetByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
}).RequireAuthorization().WithName("GetUser");

app.MapPost("/api/users", async (CreateUserRequest req, IUserService svc) => {
    var user = await svc.CreateAsync(req);
    return Results.Created($"/api/users/{user.Id}", user);
}).RequireAuthorization("Admin");

app.MapHealthChecks("/health");
app.Run();
```

---

## 7. Ruby on Rails

```ruby
# config/initializers/application.rb — production settings
Rails.application.configure do
  config.force_ssl                           = true
  config.log_level                           = :info
  config.log_formatter                       = ::Logger::Formatter.new
  config.log_tags                            = [:request_id]
  config.active_record.query_log_tags_enabled = true  # N+1 tracing
end

# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < ApplicationController
      before_action :authenticate_user!
      before_action :require_admin!, only: [:create, :destroy]
      before_action :set_user, only: [:show, :update, :destroy]

      def index
        users = User.active
                    .includes(:profile, :orders)   # eager load — prevent N+1
                    .order(created_at: :desc)
                    .page(params[:page]).per(params[:per] || 20)

        render json: UserBlueprint.render_as_hash(users, view: :list, meta: pagination_meta(users))
      end

      def create
        user = User.new(user_params)
        if user.save
          render json: UserBlueprint.render_as_hash(user), status: :created
        else
          render json: { error: { code: "VALIDATION_ERROR", details: user.errors } }, status: :unprocessable_entity
        end
      end

      private
        def set_user
          @user = User.find(params[:id])
        rescue ActiveRecord::RecordNotFound
          render json: { error: { code: "NOT_FOUND", message: "User not found" } }, status: :not_found
        end

        def user_params
          params.require(:user).permit(:name, :email, :role)
        end
    end
  end
end
```

---

## 8. Language-specific production checklist

```bash
# Node.js
node --version        # >= 20 LTS
cat package.json | python3 -c "
import json,sys; d=json.load(sys.stdin)
engines = d.get('engines',{})
print('engines:', engines)
start = d.get('scripts',{}).get('start','')
print('Start script uses node directly:', 'node' in start and 'ts-node' not in start)
"

# Python
python3 --version     # >= 3.11
grep -n "DEBUG\s*=\s*True\|debug=True" app/core/config.py 2>/dev/null

# Go
go version            # >= 1.22
go vet ./... 2>/dev/null
staticcheck ./... 2>/dev/null

# Java
java --version        # >= 21 LTS
grep "ddl-auto" src/main/resources/application.yml | grep -v validate | head -3

# Rust
cargo --version       # >= 1.77
cargo audit 2>/dev/null | head -10

# Ruby
ruby --version        # >= 3.2
bundle audit check --update 2>/dev/null | head -10
```
