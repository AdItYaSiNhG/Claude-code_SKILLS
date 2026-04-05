---
name: observability
description: >
  Use this skill for observability and SRE tasks: structured logging setup,
  distributed tracing (OpenTelemetry), metrics and alerting (Prometheus/Grafana),
  error tracking (Sentry/Rollbar), SLO/SLA definition, incident runbook generation,
  on-call escalation, chaos engineering, capacity planning, cost anomaly detection,
  and synthetic monitoring.
  Triggers: "observability", "structured logging", "tracing", "OpenTelemetry",
  "Prometheus", "Grafana", "Sentry", "error tracking", "SLO", "SLA", "incident
  runbook", "on-call", "alerting", "dashboards", "metrics", "synthetic monitoring".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Observability & SRE Skill

## Why this skill exists

You cannot fix what you cannot see. Observability done right means any production
issue can be debugged entirely from logs, traces, and metrics — without SSH access
to prod servers. This skill provides **the three pillars (logs, traces, metrics)
plus SLOs, runbooks, and chaos testing** as a cohesive system.

---

## 0. Audit current observability

```bash
# Detect observability stack
cat package.json requirements.txt 2>/dev/null | python3 - <<'EOF'
import sys
text = sys.stdin.read()
libs = {
    "Logging":   ["pino","winston","structlog","loguru","zap","zerolog"],
    "Tracing":   ["@opentelemetry","opentelemetry","jaeger","zipkin","datadog"],
    "Metrics":   ["prom-client","prometheus","statsd","datadog-metrics"],
    "Errors":    ["@sentry","sentry-sdk","rollbar","bugsnag","honeybadger"],
    "APM":       ["newrelic","datadog","elastic-apm","dynatrace"],
}
for category, items in libs.items():
    found = [i for i in items if i in text.lower()]
    if found: print(f"{category}: {found}")
EOF

# Check for existing logging patterns
grep -rn "console\.log\|print(\|logger\." \
  --include="*.ts" --include="*.py" . \
  | grep -v "node_modules\|test" | wc -l
echo "log statements found"
```

---

## 1. Structured logging

### Node.js — Pino (fastest, JSON by default)
```typescript
// src/shared/logger.ts
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  // Pretty-print in development, JSON in production
  ...(process.env.NODE_ENV !== "production"
    ? { transport: { target: "pino-pretty", options: { colorize: true } } }
    : {}),
  // Redact sensitive fields — never log these
  redact: {
    paths: ["req.headers.authorization", "*.password", "*.token", "*.secret",
            "*.creditCard", "*.ssn"],
    censor: "[REDACTED]",
  },
  // Standard fields on every log line
  base: {
    service: process.env.SERVICE_NAME ?? "api",
    version: process.env.APP_VERSION ?? "unknown",
    env:     process.env.NODE_ENV ?? "development",
  },
});

// Child logger with request context — use this in request handlers
export function requestLogger(requestId: string, userId?: string) {
  return logger.child({ requestId, userId });
}
```

```typescript
// Usage — structured, searchable, never console.log
logger.info({ userId: user.id, action: "user.created" }, "User created");
logger.error({ err: { message: err.message, code: err.code }, orderId }, "Order failed");
logger.warn({ responseTime: 850, threshold: 500, endpoint: "/api/users" }, "Slow response");

// NEVER log raw objects with potential PII
// BAD:  logger.info({ user })          — may contain password, tokens
// GOOD: logger.info({ userId: user.id }) — only the safe identifier
```

### Python — structlog
```python
# app/core/logging.py
import structlog
import logging

def configure_logging(level: str = "INFO"):
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.dev.ConsoleRenderer() if is_dev else structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelName(level)
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
    )

log = structlog.get_logger()

# Usage
log.info("user_created", user_id=user.id, email_domain=user.email.split("@")[1])
log.error("order_failed", order_id=order_id, error=str(err), error_type=type(err).__name__)
```

---

## 2. OpenTelemetry distributed tracing

```typescript
// src/telemetry/tracer.ts — initialise BEFORE importing anything else
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]:    process.env.SERVICE_NAME ?? "api",
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION ?? "0.0.0",
    environment: process.env.NODE_ENV ?? "development",
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://localhost:4318/v1/traces",
  }),
  instrumentations: [getNodeAutoInstrumentations({
    "@opentelemetry/instrumentation-fs": { enabled: false }, // too noisy
  })],
});

sdk.start();
process.on("SIGTERM", () => sdk.shutdown());

// Manual spans for important business operations
import { trace, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("myapp");

export async function withSpan<T>(
  name: string,
  attributes: Record<string, string | number>,
  fn: () => Promise<T>
): Promise<T> {
  return tracer.startActiveSpan(name, async (span) => {
    Object.entries(attributes).forEach(([k, v]) => span.setAttribute(k, v));
    try {
      const result = await fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: (err as Error).message });
      span.recordException(err as Error);
      throw err;
    } finally {
      span.end();
    }
  });
}

// Usage
const order = await withSpan("order.create", { "user.id": userId, "order.items": items.length }, () =>
  orderService.create({ userId, items })
);
```

---

## 3. Prometheus metrics (Node.js)

```typescript
// src/metrics/index.ts
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from "prom-client";

const registry = new Registry();
collectDefaultMetrics({ register: registry });

// HTTP request metrics
export const httpRequestDuration = new Histogram({
  name:    "http_request_duration_seconds",
  help:    "HTTP request duration in seconds",
  labelNames: ["method", "route", "status_code"],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [registry],
});

export const httpRequestTotal = new Counter({
  name:    "http_requests_total",
  help:    "Total HTTP requests",
  labelNames: ["method", "route", "status_code"],
  registers: [registry],
});

// Business metrics
export const ordersCreated = new Counter({
  name:    "orders_created_total",
  help:    "Total orders created",
  labelNames: ["tier", "channel"],
  registers: [registry],
});

export const activeUsers = new Gauge({
  name:    "active_users_current",
  help:    "Currently active users (last 5 min)",
  registers: [registry],
});

// Expose metrics endpoint
app.get("/metrics", async (req, res) => {
  res.set("Content-Type", registry.contentType);
  res.send(await registry.metrics());
});

// Express middleware to auto-record request metrics
export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const end = httpRequestDuration.startTimer();
  res.on("finish", () => {
    const labels = { method: req.method, route: req.route?.path ?? "unknown", status_code: res.statusCode };
    end(labels);
    httpRequestTotal.inc(labels);
  });
  next();
}
```

---

## 4. SLO definition

```yaml
# slos/api-slos.yaml — define before configuring alerts

service: orders-api

slos:
  - name: availability
    description: "API responds to requests successfully"
    target: 99.9   # 99.9% = ~8.7 hours downtime/year
    window: 30d
    indicator:
      type: request_based
      good_events: "http_requests_total{status_code!~'5..'}"
      total_events: "http_requests_total"

  - name: latency_p95
    description: "95th percentile response time under 500ms"
    target: 95.0
    window: 30d
    indicator:
      type: request_based
      good_events: "http_request_duration_seconds_bucket{le='0.5'}"
      total_events: "http_request_duration_seconds_count"

  - name: error_rate
    description: "Less than 1% of requests result in errors"
    target: 99.0
    window: 7d
    indicator:
      type: request_based
      good_events: "http_requests_total{status_code!~'5..'}"
      total_events: "http_requests_total"
```

---

## 5. Incident runbook template

```bash
mkdir -p docs/runbooks

cat > docs/runbooks/high-error-rate.md << 'EOF'
# Runbook: High Error Rate

**Alert:** `error_rate > 5% for 5 minutes`
**Severity:** P1
**Owner:** Platform team
**Last updated:** $(date +%Y-%m-%d)

## 1. Immediate triage (< 5 min)

```bash
# Check current error rate
curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status_code=~'5..'}[5m])" | python3 -m json.tool

# Check recent deployments
kubectl rollout history deployment/api -n prod | tail -5

# Check error logs
kubectl logs -l app=api -n prod --since=10m | grep '"level":"error"' | tail -20
```

## 2. Identify the cause

- [ ] Check if error started at a deployment (rollback if yes — see section 4)
- [ ] Check if error is on a specific endpoint: `grep "route" logs | sort | uniq -c | sort -rn`
- [ ] Check DB connectivity: `kubectl exec -it api-pod -- curl db-service:5432`
- [ ] Check downstream services: `kubectl exec -it api-pod -- curl payment-service/health`

## 3. Escalation

If unresolved after 15 minutes:
- Notify: #incidents Slack channel
- Page: On-call engineer via PagerDuty

## 4. Rollback

```bash
kubectl rollout undo deployment/api -n prod
kubectl rollout status deployment/api -n prod
# Verify: watch curl -s https://api.myapp.com/health
```

## 5. Post-incident

- [ ] File incident report within 24h
- [ ] Add regression test for root cause
- [ ] Update this runbook if process was unclear
EOF
```

---

## 6. Sentry error tracking

```typescript
// src/telemetry/sentry.ts
import * as Sentry from "@sentry/node";

Sentry.init({
  dsn:              process.env.SENTRY_DSN,
  environment:      process.env.NODE_ENV,
  release:          process.env.APP_VERSION,
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.1 : 1.0,
  // Don't send noise
  ignoreErrors: [
    "NotFoundError",       // expected — user navigated to wrong URL
    "UnauthorizedError",   // expected — invalid tokens
    /ResizeObserver/,      // browser quirk, not actionable
  ],
  beforeSend: (event) => {
    // Strip PII before sending to Sentry
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    return event;
  },
});

// Capture with context
export function captureError(err: Error, context: Record<string, unknown> = {}) {
  Sentry.withScope(scope => {
    Object.entries(context).forEach(([k, v]) => scope.setExtra(k, v));
    Sentry.captureException(err);
  });
}
```

---

## 7. Synthetic monitoring

```typescript
// monitoring/synthetic/api-check.ts — runs every minute in prod
import https from "https";

const CHECKS = [
  { name: "health", url: "https://api.myapp.com/health", expectedStatus: 200, maxMs: 500 },
  { name: "auth",   url: "https://api.myapp.com/api/v1/me",  expectedStatus: 401, maxMs: 300 },
];

async function runChecks() {
  for (const check of CHECKS) {
    const start = Date.now();
    try {
      const res = await fetch(check.url, { signal: AbortSignal.timeout(5000) });
      const latency = Date.now() - start;
      const ok = res.status === check.expectedStatus && latency < check.maxMs;
      console.log(JSON.stringify({
        check: check.name, status: res.status,
        latency_ms: latency, ok,
        timestamp: new Date().toISOString(),
      }));
      if (!ok) {
        // Alert via PagerDuty / Slack webhook
        await alertOnCall(check.name, res.status, latency);
      }
    } catch (err) {
      console.error(JSON.stringify({ check: check.name, error: (err as Error).message, ok: false }));
    }
  }
}

runChecks();
```
