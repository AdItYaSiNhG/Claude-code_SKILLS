---
name: devops-observability
description: Production observability, SLO/SLA definition, alerting strategy, distributed tracing, and incident runbooks. Use for setting up monitoring stacks (Prometheus/Grafana, Datadog, OpenTelemetry), defining error budgets, and structuring incident response.
---

The user is setting up observability or managing a production incident. Apply the relevant section.

## Observability Pillars

```
Metrics  → What is happening? (rates, errors, durations, saturation)
Logs     → Why is it happening? (structured context per event)
Traces   → Where is it happening? (latency across service boundaries)
Profiles → How is CPU/memory spent? (flamegraphs, continuous profiling)
```

Always instrument all four. Metrics for alerting, traces for debugging, logs for evidence, profiles for optimisation.

## SLO Definition

### SLO framework (Google SRE)
```
SLI (indicator)  → the metric: e.g. "fraction of requests returning 2xx in < 500ms"
SLO (objective)  → the target:  e.g. "99.5% of requests over 30-day window"
SLA (agreement)  → the contract: SLO minus buffer (e.g. 99.0% for customer SLA)
Error budget     → 1 - SLO = 0.5% of requests = ~216 minutes downtime/month
```

### Recommended SLOs by tier

| Service tier | Availability SLO | Latency SLO (p99) | Error budget / month |
|---|---|---|---|
| Revenue-critical (checkout, auth) | 99.9% | < 1s | 43 min |
| Core product (dashboard, API) | 99.5% | < 2s | 3.6 hrs |
| Internal tools | 99.0% | < 5s | 7.2 hrs |
| Batch / async | 99.0% (success rate) | p99 < 30s | — |

### SLO as code (Prometheus recording rules)
```yaml
# prometheus/rules/slo-orders-api.yml
groups:
  - name: slo_orders_api
    rules:
      # SLI: successful requests
      - record: slo:requests:success_rate5m
        expr: |
          sum(rate(http_requests_total{service="orders-api",code=~"2.."}[5m]))
          /
          sum(rate(http_requests_total{service="orders-api"}[5m]))

      # Error budget burn rate (1h window)
      - record: slo:error_budget:burn_rate1h
        expr: |
          1 - slo:requests:success_rate5m
          /
          (1 - 0.995)   # 1 - SLO target
```

## Alerting Strategy (Multi-window, Multi-burn-rate)

```yaml
# Avoids alert fatigue: only page when budget is burning fast enough to exhaust within SLO window
groups:
  - name: orders-api-slo-alerts
    rules:
      # Page immediately: burning at 14× (exhausts budget in ~2hrs)
      - alert: OrdersAPIErrorBudgetCritical
        expr: slo:error_budget:burn_rate1h > 14
        for: 2m
        labels: { severity: page }
        annotations:
          summary: "Orders API error budget burning fast — paging on-call"

      # Slack alert: burning at 6× (exhausts budget in ~5hrs)
      - alert: OrdersAPIErrorBudgetWarning
        expr: slo:error_budget:burn_rate1h > 6
        for: 15m
        labels: { severity: slack }

      # Latency SLO
      - alert: OrdersAPILatencyHigh
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="orders-api"}[5m])) > 1
        for: 5m
        labels: { severity: page }
```

### Alert routing (Alertmanager / PagerDuty)
```yaml
route:
  receiver: slack-default
  routes:
    - match: { severity: page }
      receiver: pagerduty-oncall
      repeat_interval: 30m
    - match: { severity: slack }
      receiver: slack-engineering
      repeat_interval: 4h
```

## OpenTelemetry Instrumentation

### Auto-instrumentation (Node.js)
```ts
// otel.ts — load before any application code
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'

const sdk = new NodeSDK({
  serviceName: 'orders-api',
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
})
sdk.start()
```

### Manual span instrumentation
```ts
import { trace, SpanStatusCode } from '@opentelemetry/api'
const tracer = trace.getTracer('orders-api')

async function processOrder(orderId: string) {
  return tracer.startActiveSpan('processOrder', async (span) => {
    span.setAttribute('order.id', orderId)
    try {
      const result = await doWork(orderId)
      span.setStatus({ code: SpanStatusCode.OK })
      return result
    } catch (err) {
      span.recordException(err as Error)
      span.setStatus({ code: SpanStatusCode.ERROR })
      throw err
    } finally {
      span.end()
    }
  })
}
```

## Structured Logging

```ts
// Always log as JSON with consistent fields
logger.info({
  event: 'order.confirmed',
  orderId: order.id,
  customerId: order.customerId,
  totalCents: order.totalCents,
  durationMs: Date.now() - startTime,
  traceId: trace.getActiveSpan()?.spanContext().traceId,
})

// Never: logger.info(`Order ${orderId} confirmed for ${email}`)
// Never log: passwords, tokens, PII (email, phone, address)
```

## Monitoring Stack Options

| Stack | Best for |
|---|---|
| Prometheus + Grafana + Loki + Tempo | Self-hosted, cost-efficient, full OTel support |
| Datadog | Managed, fast setup, AI-powered insights, expensive at scale |
| Grafana Cloud | Managed Prometheus/Loki/Tempo, generous free tier |
| AWS CloudWatch | AWS-native, lowest friction for AWS workloads |
| New Relic | Application-centric, good for APM-first teams |

## Incident Runbook Template

```markdown
# Incident: [Service] [Symptom]
**Severity**: P1 / P2 / P3
**SLO impact**: Yes/No — budget consumed: X%

## Detection
- Alert: [alert name and link]
- Dashboards: [Grafana link]
- Traces: [Tempo/Jaeger link]

## Triage (first 5 minutes)
1. Check error rate: `sum(rate(http_requests_total{code=~"5.."}[5m]))` by service
2. Check latency p99: `histogram_quantile(0.99, ...)`
3. Check recent deploys: `kubectl rollout history deployment/orders-api`
4. Check dependency health: [Postgres, Redis, external APIs]

## Mitigation Options
- **Rollback deploy**: `kubectl rollout undo deployment/orders-api`
- **Scale up**: `kubectl scale deployment orders-api --replicas=10`
- **Enable maintenance mode**: `curl -X POST https://admin.internal/maintenance`
- **Disable feature flag**: [LaunchDarkly / flagsmith link]

## Resolution Checklist
- [ ] Error rate back to baseline
- [ ] Latency p99 within SLO
- [ ] No customer-visible impact remaining
- [ ] Stakeholders notified

## Post-Incident (within 48hrs)
- Timeline of events
- Root cause (5 Whys)
- Action items (with owners and due dates)
- SLO impact and error budget consumed
```

## Output Format

1. For SLO setup: show the recording rules, alert rules, and error budget calculation
2. For instrumentation: show the actual SDK code for their stack (not pseudocode)
3. For incidents: use the runbook template, pre-filled with their service details
4. Always include the Grafana/Datadog query for the key metric being discussed
