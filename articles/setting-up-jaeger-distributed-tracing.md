---
layout: default
title: "Setting Up Jaeger for Distributed Tracing"
description: "Deploy Jaeger for distributed tracing across microservices with OpenTelemetry instrumentation, Elasticsearch storage, and Grafana dashboards"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-jaeger-distributed-tracing/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Distributed tracing shows you where time goes across service boundaries. Jaeger collects OpenTelemetry spans and lets your team trace a request from API gateway through microservices to database. This guide deploys Jaeger all-in-one for development and a production-ready setup with Elasticsearch for persistence.

## Table of Contents

- [Development: Jaeger All-in-One](#development-jaeger-all-in-one)
- [Production: Docker Compose with Elasticsearch](#production-docker-compose-with-elasticsearch)
- [Instrumenting a Python Service](#instrumenting-a-python-service)
- [Instrumenting a Node.js Service](#instrumenting-a-nodejs-service)
- [Manual Span Creation](#manual-span-creation)
- [Jaeger Query API for Automation](#jaeger-query-api-for-automation)
- [Grafana Integration](#grafana-integration)
- [Trace Sampling Configuration](#trace-sampling-configuration)
- [Trace Retention and Index Lifecycle Management](#trace-retention-and-index-lifecycle-management)
- [Adding Context Propagation Across Queues](#adding-context-propagation-across-queues)
- [Kubernetes Deployment with Jaeger Operator](#kubernetes-deployment-with-jaeger-operator)
- [Alerting on Trace Anomalies](#alerting-on-trace-anomalies)
- [Correlating Traces with Logs](#correlating-traces-with-logs)

## Development: Jaeger All-in-One

```bash
# Quick start for development — all components in one container
docker run -d \
  --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \   # Jaeger UI
  -p 14250:14250 \
  -p 14268:14268 \
  -p 14269:14269 \
  -p 4317:4317 \     # OTLP gRPC
  -p 4318:4318 \     # OTLP HTTP
  --restart unless-stopped \
  jaegertracing/all-in-one:1.55

# Open Jaeger UI
open http://localhost:16686
```

## Production: Docker Compose with Elasticsearch

```yaml
# docker-compose.yml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1
    restart: unless-stopped
    networks:
      - tracing

  jaeger-collector:
    image: jaegertracing/jaeger-collector:1.55
    container_name: jaeger-collector
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - COLLECTOR_OTLP_ENABLED=true
      - LOG_LEVEL=info
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "14268:14268" # HTTP collector
      - "14250:14250" # gRPC collector
    depends_on:
      - elasticsearch
    restart: unless-stopped
    networks:
      - tracing

  jaeger-query:
    image: jaegertracing/jaeger-query:1.55
    container_name: jaeger-query
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - LOG_LEVEL=info
    ports:
      - "16686:16686"  # Jaeger UI
      - "16687:16687"  # Admin port
    depends_on:
      - elasticsearch
    restart: unless-stopped
    networks:
      - tracing

networks:
  tracing:
    driver: bridge

volumes:
  es_data:
```

```bash
docker compose up -d
# Wait for Elasticsearch to start (~30 seconds)
docker compose logs -f jaeger-collector
```

## Instrumenting a Python Service

```bash
pip install opentelemetry-api opentelemetry-sdk \
  opentelemetry-exporter-otlp-proto-grpc \
  opentelemetry-instrumentation-fastapi \
  opentelemetry-instrumentation-httpx \
  opentelemetry-instrumentation-sqlalchemy
```

```python
# tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

def configure_tracing(service_name: str, otlp_endpoint: str = "http://jaeger-collector:4317"):
    resource = Resource(attributes={
        SERVICE_NAME: service_name,
        "service.version": "1.0.0",
        "deployment.environment": "production",
    })

    provider = TracerProvider(resource=resource)

    otlp_exporter = OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
    provider.add_span_processor(BatchSpanProcessor(otlp_exporter))

    trace.set_tracer_provider(provider)

    # Auto-instrument frameworks
    FastAPIInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()

    return trace.get_tracer(service_name)
```

```python
# main.py
from fastapi import FastAPI
from tracing import configure_tracing
import httpx

app = FastAPI()
tracer = configure_tracing("order-service")

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    # Manual span for custom operations
    with tracer.start_as_current_span("fetch-order-details") as span:
        span.set_attribute("order.id", order_id)

        # This HTTP call will be auto-instrumented
        async with httpx.AsyncClient() as client:
            user = await client.get(f"http://user-service/users/{order_id}")

        span.set_attribute("user.id", user.json()["id"])
        return {"order_id": order_id, "user": user.json()}
```

## Instrumenting a Node.js Service

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-grpc
```

```javascript
// tracing.js (must be required before other modules)
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'payment-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '2.1.0',
    'deployment.environment': process.env.NODE_ENV || 'development',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTLP_ENDPOINT || 'http://jaeger-collector:4317',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false }, // Too noisy
    }),
  ],
});

sdk.start();
process.on('SIGTERM', () => sdk.shutdown());
```

```json
// package.json start script
{
  "scripts": {
    "start": "node -r ./tracing.js server.js"
  }
}
```

## Manual Span Creation

```python
# Python: add custom spans for business logic
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_payment(payment_id: str, amount: float):
    with tracer.start_as_current_span("process-payment") as span:
        span.set_attribute("payment.id", payment_id)
        span.set_attribute("payment.amount", amount)
        span.set_attribute("payment.currency", "USD")

        try:
            result = charge_card(payment_id, amount)
            span.set_attribute("payment.status", "success")
            span.set_attribute("payment.transaction_id", result.transaction_id)
            return result
        except PaymentDeclinedException as e:
            span.set_attribute("payment.status", "declined")
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            raise
```

## Jaeger Query API for Automation

```bash
# Find traces with errors
curl "http://localhost:16686/api/traces?service=order-service&tags=%7B%22error%22%3A%22true%22%7D&limit=20"

# Get trace by ID
curl "http://localhost:16686/api/traces/abcdef1234567890"

# List all services
curl "http://localhost:16686/api/services"

# Search slow traces (>1 second)
curl "http://localhost:16686/api/traces?service=order-service&minDuration=1000ms&limit=50"
```

## Grafana Integration

Add Jaeger as a Grafana data source:

```yaml
# grafana/provisioning/datasources/jaeger.yml
apiVersion: 1

datasources:
  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger-query:16686
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        filterByTraceID: true
        mapTagNamesEnabled: true
        mappedTags:
          - key: service.name
            value: service
      nodeGraph:
        enabled: true
```

In Grafana dashboards, add a trace panel:

```
Panel Type: Traces
Data Source: Jaeger
Query: { service="order-service" }
```

## Trace Sampling Configuration

For high-traffic production services, sample selectively:

```python
from opentelemetry.sdk.trace.sampling import (
    TraceIdRatioBased,
    ParentBased,
    ALWAYS_ON,
    ALWAYS_OFF,
)

# Sample 10% of traces in production
sampler = ParentBased(
    root=TraceIdRatioBased(0.1),
    remote_parent_sampled=ALWAYS_ON,    # Always sample if parent was sampled
    remote_parent_not_sampled=ALWAYS_OFF,
)

provider = TracerProvider(resource=resource, sampler=sampler)
```

## Trace Retention and Index Lifecycle Management

Jaeger with Elasticsearch accumulates data quickly. A busy service generating 1,000 traces per minute fills tens of gigabytes per day. Configure an ILM policy in Elasticsearch to roll over and delete old trace indices automatically:

```bash
# Create ILM policy for Jaeger span indices
curl -X PUT "http://localhost:9200/_ilm/policy/jaeger-span-policy" \
  -H 'Content-Type: application/json' \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_age": "1d",
              "max_size": "10gb"
            }
          }
        },
        "warm": {
          "min_age": "3d",
          "actions": {
            "shrink": { "number_of_shards": 1 },
            "forcemerge": { "max_num_segments": 1 }
          }
        },
        "delete": {
          "min_age": "14d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

This keeps 14 days of traces and aggressively merges warm shards after 3 days to reduce heap pressure. Adjust `min_age` under `delete` based on your incident response SLA — most teams find 7-30 days sufficient.

Jaeger also ships a `jaeger-es-index-cleaner` utility for simpler retention without ILM:

```bash
# Delete indices older than 14 days
docker run --rm \
  -e ROLLOVER=true \
  jaegertracing/jaeger-es-index-cleaner:1.55 \
  14 http://elasticsearch:9200
```

Schedule this as a cron job or a daily Docker Compose service to keep storage bounded without configuring full ILM.

## Adding Context Propagation Across Queues

Auto-instrumentation handles HTTP calls automatically, but message queues require explicit context propagation. Here is a pattern for RabbitMQ using the W3C TraceContext format:

```python
# Producer: inject trace context into message headers
from opentelemetry import trace, propagate
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

def publish_order_event(channel, order_id: str, payload: dict):
    tracer = trace.get_tracer(__name__)

    with tracer.start_as_current_span("publish-order-event") as span:
        span.set_attribute("messaging.system", "rabbitmq")
        span.set_attribute("messaging.destination", "orders")
        span.set_attribute("order.id", order_id)

        headers = {}
        propagate.inject(headers)  # Injects traceparent + tracestate

        channel.basic_publish(
            exchange="orders",
            routing_key="order.created",
            body=json.dumps(payload),
            properties=pika.BasicProperties(headers=headers)
        )
```

```python
# Consumer: extract trace context from message headers
def process_message(channel, method, properties, body):
    tracer = trace.get_tracer(__name__)
    ctx = propagate.extract(properties.headers or {})

    with tracer.start_as_current_span(
        "process-order-event",
        context=ctx,
        kind=trace.SpanKind.CONSUMER
    ) as span:
        span.set_attribute("messaging.system", "rabbitmq")
        order = json.loads(body)
        handle_order(order)
```

This ensures that a trace from the HTTP request that triggered the publish appears connected to the consumer span in Jaeger — giving you end-to-end visibility across the queue boundary.

## Kubernetes Deployment with Jaeger Operator

For Kubernetes environments, the Jaeger Operator simplifies lifecycle management:

```bash
# Install cert-manager (prerequisite)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Install Jaeger Operator
kubectl create namespace observability
kubectl apply -n observability -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.55.0/jaeger-operator.yaml
```

```yaml
# jaeger-production.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
  collector:
    replicas: 2
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
  query:
    replicas: 1
    resources:
      limits:
        cpu: 250m
        memory: 256Mi
  ingress:
    enabled: true
    hosts:
      - tracing.example.com
```

The operator manages rolling updates and handles schema migration for Elasticsearch indices automatically, which is a significant operational advantage over managing the collector and query components separately.

## Alerting on Trace Anomalies

Jaeger itself does not ship alerting, but you can build lightweight trace-based alerts by querying the Jaeger HTTP API from a scheduled script and routing results to PagerDuty or Slack. A practical pattern for remote teams:

```python
#!/usr/bin/env python3
# scripts/jaeger-alert.py — run every 5 minutes via cron
import requests, json, os, sys
from datetime import datetime, timedelta

JAEGER_URL = os.getenv("JAEGER_URL", "http://localhost:16686")
SLACK_WEBHOOK = os.getenv("SLACK_WEBHOOK")
ERROR_THRESHOLD = int(os.getenv("ERROR_THRESHOLD", "10"))
SERVICES = ["order-service", "payment-service", "user-service"]

now = datetime.utcnow()
five_min_ago = now - timedelta(minutes=5)
start_us = int(five_min_ago.timestamp() * 1_000_000)
end_us   = int(now.timestamp() * 1_000_000)

alerts = []
for service in SERVICES:
    resp = requests.get(
        f"{JAEGER_URL}/api/traces",
        params={
            "service": service,
            "tags": '{"error":"true"}',
            "start": start_us,
            "end": end_us,
            "limit": 100,
        },
        timeout=10,
    )
    traces = resp.json().get("data", [])
    if len(traces) >= ERROR_THRESHOLD:
        alerts.append(f"*{service}*: {len(traces)} error traces in the last 5 minutes")

if alerts and SLACK_WEBHOOK:
    payload = {"text": "Jaeger trace alert:\n" + "\n".join(alerts)}
    requests.post(SLACK_WEBHOOK, json=payload, timeout=5)
    sys.exit(1)

print("No anomalies detected.")
```

Add this to a cron job on your monitoring host or run it as a Kubernetes CronJob. It keeps alerting logic simple and avoids the complexity of a full APM platform for teams that only need error rate signals from traces.

## Correlating Traces with Logs

The highest-value Jaeger integration for most teams is log correlation: clicking a span in Jaeger and jumping directly to the logs that span generated, without copying trace IDs manually. This requires two things: your services must include the trace ID in log output, and Grafana must link the Jaeger trace ID to Loki.

Inject the trace ID into structured logs automatically using the OpenTelemetry logging bridge:

```python
# logging_config.py
import logging
from opentelemetry._logs import set_logger_provider
from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter

def configure_logging(service_name: str):
    logger_provider = LoggerProvider()
    otlp_exporter = OTLPLogExporter(endpoint="http://jaeger-collector:4317", insecure=True)
    logger_provider.add_log_record_processor(BatchLogRecordProcessor(otlp_exporter))
    set_logger_provider(logger_provider)

    handler = LoggingHandler(level=logging.DEBUG, logger_provider=logger_provider)

    # Add trace context fields to every log record
    logging.basicConfig(handlers=[handler], level=logging.INFO)
    return logging.getLogger(service_name)
```

With this in place, every `logger.info(...)` call automatically includes `trace_id` and `span_id` fields in the OTLP payload. Loki receives these via a Promtail pipeline, and Grafana's trace-to-logs linking uses the `trace_id` field to jump between the two data sources with a single click.

---

## Related Articles

- [GitHub Pull Request Workflow for Distributed Teams](/remote-work-tools/github-pull-request-workflow-for-distributed-teams/)
- [Best Time Zone Management Tools for Distributed Engineering](/remote-work-tools/best-time-zone-management-tools-for-distributed-engineering-teams-2026/)
- [How to Run Remote Developer Hackathon for Distributed](/remote-work-tools/how-to-run-remote-developer-hackathon-for-distributed-engine/)
- [Best API Key Management Workflow for Remote Development](/remote-work-tools/best-api-key-management-workflow-for-remote-development-team/)
- [Remote Legal Research Tool Comparison for Distributed Law](/remote-work-tools/remote-legal-research-tool-comparison-for-distributed-law-fi/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
