---
name: monitoring-observability
description: Making a service observable — OpenTelemetry traces and metrics, Prometheus, structured logs, and SLO-based alerting
---

# Monitoring and observability

## Wiring up OpenTelemetry

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";
import { PgInstrumentation } from "@opentelemetry/instrumentation-pg";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";

const sdk = new NodeSDK({
  serviceName: "order-service",
  traceExporter: new OTLPTraceExporter({
    url: "http://otel-collector:4318/v1/traces",
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: "http://otel-collector:4318/v1/metrics",
    }),
    exportIntervalMillis: 15000,
  }),
  instrumentations: [
    new HttpInstrumentation(),
    new PgInstrumentation(),
  ],
});

sdk.start();
process.on("SIGTERM", () => sdk.shutdown());
```

## Custom spans and metrics

```typescript
import { trace, metrics, SpanStatusCode } from "@opentelemetry/api";

const tracer = trace.getTracer("order-service");
const meter = metrics.getMeter("order-service");

const orderCounter = meter.createCounter("orders.created", {
  description: "Number of orders created",
});

const orderDuration = meter.createHistogram("orders.processing_duration_ms", {
  description: "Order processing duration in milliseconds",
  unit: "ms",
});

async function createOrder(input: CreateOrderInput) {
  return tracer.startActiveSpan("createOrder", async (span) => {
    try {
      span.setAttributes({
        "order.customer_id": input.customerId,
        "order.item_count": input.items.length,
      });

      const start = performance.now();
      const order = await db.order.create({ data: input });

      orderCounter.add(1, { status: "success" });
      orderDuration.record(performance.now() - start);

      span.setStatus({ code: SpanStatusCode.OK });
      return order;
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      orderCounter.add(1, { status: "error" });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

## Exposing Prometheus metrics

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "api-servers"
    static_configs:
      - targets: ["api-1:9090", "api-2:9090"]
    metrics_path: /metrics

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

```typescript
import { collectDefaultMetrics, Counter, Histogram, Registry } from "prom-client";

const registry = new Registry();
collectDefaultMetrics({ register: registry });

const httpRequestDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [registry],
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on("finish", () => {
    end({ method: req.method, route: req.route?.path ?? req.path, status: res.statusCode });
  });
  next();
});

app.get("/metrics", async (req, res) => {
  res.set("Content-Type", registry.contentType);
  res.end(await registry.metrics());
});
```

## Structured logging

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  formatters: {
    level: (label) => ({ level: label }),
  },
  redact: ["req.headers.authorization", "password", "token"],
});

function requestLogger(req, res, next) {
  const start = Date.now();
  res.on("finish", () => {
    logger.info({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration_ms: Date.now() - start,
      trace_id: req.headers["x-trace-id"],
    });
  });
  next();
}
```

## Alerting rules

```yaml
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_request_duration_seconds_count{status=~"5.."}[5m]) / rate(http_request_duration_seconds_count[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5% for {{ $labels.route }}"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
```

## What to avoid

- Logging secrets (passwords, tokens, PII) with no redaction
- Interpolating values into log strings instead of using structured fields
- Unbounded label cardinality — user IDs as metric labels, for instance
- Logs and traces that can't be joined by a shared trace ID
- Alerting on symptoms like high CPU without tying them to a cause
- Building dashboards before you've defined any SLOs

## Before you ship

- [ ] The OpenTelemetry SDK auto-instruments HTTP, DB, and messaging
- [ ] Business-critical operations carry custom spans
- [ ] Metric labels stay bounded in cardinality
- [ ] Logs are JSON-structured with secrets redacted
- [ ] Trace context propagates across service boundaries
- [ ] Alerts are derived from SLOs (error rate, latency percentiles)
- [ ] Dashboards show RED metrics (Rate, Errors, Duration) per service
- [ ] Log retention and rotation policies are in place
