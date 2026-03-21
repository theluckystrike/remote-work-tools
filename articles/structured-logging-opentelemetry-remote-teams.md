---
layout: default
title: "Structured Logging and OpenTelemetry for Remote Teams"
description: "Set up structured logging and distributed tracing with OpenTelemetry for remote engineering teams. Covers log formats, trace propagation, exporters, and Grafana Tempo."
date: 2026-03-21
author: theluckystrike
permalink: /structured-logging-opentelemetry-remote-teams/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

Remote teams debugging production issues need observability infrastructure that works without being in the same room. Structured logging and distributed tracing are the two tools that make it possible to answer "what happened and why" from anywhere, asynchronously.

This guide covers structured logging setup, OpenTelemetry trace instrumentation, and exporting traces and logs to a self-hosted stack using Grafana Tempo and Loki.

## Why Structured Logging Over Plain Text

Plain text logs require grep patterns to extract information. Structured logs are queryable like a database.

```
# Plain text log — hard to query at scale
2026-03-21 14:23:01 ERROR Failed to process payment for user 12345: timeout after 5000ms

# Structured log (JSON) — every field is queryable
{
  "timestamp": "2026-03-21T14:23:01.234Z",
  "level": "error",
  "message": "payment processing failed",
  "user_id": "12345",
  "error": "timeout",
  "timeout_ms": 5000,
  "service": "payment-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

With structured logs, a query like "all errors for user 12345 in the payment service in the last hour" takes seconds. With plain text, it requires brittle regex.

## Node.js: Pino Structured Logger

```bash
npm install pino pino-pretty
```

```javascript
// logger.js
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // In production, output JSON (no pretty printing — it's slow)
  // In development, use pino-pretty for human-readable output
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  formatters: {
    level(label) {
      return { level: label };
    },
  },
  base: {
    service: process.env.SERVICE_NAME || 'unknown-service',
    env: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

export default logger;
```

```javascript
// Usage in request handler
import logger from './logger.js';

const childLogger = logger.child({
  request_id: req.headers['x-request-id'],
  user_id: req.user?.id,
});

childLogger.info({ action: 'checkout_started', cart_total: cart.total }, 'Checkout initiated');

try {
  await processPayment(cart);
  childLogger.info({ action: 'payment_success' }, 'Payment completed');
} catch (err) {
  childLogger.error({ err, action: 'payment_failed' }, 'Payment processing error');
  throw err;
}
```

## Python: structlog

```bash
pip install structlog
```

```python
# logging_config.py
import logging
import structlog

def configure_logging():
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.stdlib.add_log_level,
            structlog.stdlib.add_logger_name,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer() if not os.getenv("DEV_MODE")
            else structlog.dev.ConsoleRenderer(),
        ],
        wrapper_class=structlog.BoundLogger,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

# Usage
import structlog

log = structlog.get_logger()

# Bind context for the lifetime of a request
structlog.contextvars.bind_contextvars(
    request_id=request.id,
    user_id=user.id,
    service="api-gateway",
)

log.info("payment_started", cart_total=cart.total, item_count=len(cart.items))
```

## OpenTelemetry: Add Distributed Tracing

OpenTelemetry is the standard for distributed tracing. It propagates trace context across service boundaries so you can follow a request through multiple services.

```bash
# Node.js
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-otlp-grpc

# Python
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

```javascript
// otel.js — initialize before importing your app code
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-otlp-grpc';
import { resourceFromAttributes } from '@opentelemetry/resources';

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    'service.name': process.env.SERVICE_NAME || 'my-service',
    'service.version': process.env.APP_VERSION || '0.0.0',
    'deployment.environment': process.env.NODE_ENV || 'development',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4317',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
    }),
  ],
});

sdk.start();
```

```javascript
// Add custom spans for business logic
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('payment-service');

async function processPayment(cart) {
  return tracer.startActiveSpan('processPayment', async (span) => {
    span.setAttribute('cart.total', cart.total);
    span.setAttribute('cart.item_count', cart.items.length);

    try {
      const result = await chargeCard(cart);
      span.setAttribute('payment.status', 'success');
      span.setAttribute('payment.transaction_id', result.transactionId);
      return result;
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

## Self-Hosted Observability Stack

```yaml
# docker-compose.observability.yml
version: "3.9"

services:
  # OpenTelemetry Collector — receives traces, sends to Tempo
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.98.0
    ports:
      - "4317:4317"   # gRPC
      - "4318:4318"   # HTTP
    volumes:
      - ./otel-config.yml:/etc/otel-collector-config.yml:ro
    command: ["--config=/etc/otel-collector-config.yml"]

  # Grafana Tempo — distributed trace storage
  tempo:
    image: grafana/tempo:2.4.1
    ports:
      - "3200:3200"
    volumes:
      - ./tempo.yml:/etc/tempo.yml:ro
      - tempo_data:/tmp/tempo
    command: ["-config.file=/etc/tempo.yml"]

  # Grafana Loki — log aggregation
  loki:
    image: grafana/loki:2.9.7
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki

  # Grafana — dashboards for traces + logs
  grafana:
    image: grafana/grafana:10.4.0
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin

volumes:
  tempo_data: {}
  loki_data: {}
```

```yaml
# otel-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
```

## Correlate Traces and Logs

The trace_id and span_id from OpenTelemetry should appear in every log line so you can jump from a log to the corresponding trace.

```javascript
// In Pino, inject trace context automatically
import { context, trace } from '@opentelemetry/api';

const loggingMiddleware = (req, res, next) => {
  const span = trace.getActiveSpan();
  const spanContext = span?.spanContext();

  req.log = logger.child({
    trace_id: spanContext?.traceId,
    span_id: spanContext?.spanId,
    request_id: req.headers['x-request-id'],
  });

  next();
};
```

In Grafana, set up a data link from Loki log lines to Tempo traces using `trace_id`:

```
Grafana → Loki data source → Derived Fields
Field name: trace_id
Regex: "trace_id":"(\w+)"
URL: /explore?orgId=1&left=...&right={"datasource":"Tempo","queries":[{"query":"${__value.raw}"}]}
```

Clicking a `trace_id` in a log line opens the full distributed trace in Grafana Tempo — no copying and pasting.

## Related Reading

- [Prometheus Monitoring Setup for Remote Infrastructure](/remote-work-tools/prometheus-monitoring-remote-infrastructure/)
- [CI/CD Pipeline for Solo Developers: GitHub Actions](/remote-work-tools/ci-cd-pipeline-solo-developer-github-actions/)
- [Home Lab Setup Guide for Remote Developers](/remote-work-tools/home-lab-setup-guide-remote-developers/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
