---
layout: default
title: "How to Set Up Vector for Log Processing"
description: "Deploy Vector as a high-performance log aggregation and processing pipeline to collect, transform, and route logs for distributed remote teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-vector-for-log-processing/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Vector is a high-performance observability pipeline written in Rust. It collects logs, metrics, and traces, transforms them, and routes them to any destination. Compared to Fluentd and Logstash, it uses significantly less memory and CPU — important when running on the same hosts as your application. A single Vector process replaces multiple agents.

---

## Install Vector

```bash
# macOS
brew install vector

# Debian/Ubuntu
curl -1sLf 'https://repositories.timber.io/public/vector/cfg/setup/bash.deb.sh' \
  | sudo bash
sudo apt install vector

# RPM (RHEL/Rocky/Alma)
curl -1sLf 'https://repositories.timber.io/public/vector/cfg/setup/bash.rpm.sh' \
  | sudo bash
sudo dnf install vector

# Docker
docker pull timberio/vector:latest-distroless-libc
```

---

## Architecture: Agent vs Aggregator

Vector runs in two modes:

**Agent** (on each host): Collects local logs and forwards to the aggregator or directly to storage.

**Aggregator** (central node): Receives from all agents, applies heavy transforms, routes to destinations.

For small teams (< 20 services), run Vector directly on each host sending to a destination. For larger setups, use the agent/aggregator pattern.

---

## Basic Configuration

Vector's config is TOML or YAML. A complete pipeline that collects Docker logs and sends to Elasticsearch:

```toml
# /etc/vector/vector.toml

# ========================
# SOURCES
# ========================

[sources.docker_logs]
type = "docker_logs"
# Collect all container logs
include_containers = ["*"]
# Or filter:
# include_containers = ["nginx", "api", "worker"]
exclude_containers = ["vector"]  # Don't collect Vector's own logs
include_labels = {}

[sources.journald]
type = "journald"
include_units = ["nginx.service", "postgresql.service"]

[sources.http_input]
type = "http_server"
address = "0.0.0.0:8686"
encoding = "json"

# ========================
# TRANSFORMS
# ========================

# Parse JSON logs from application containers
[transforms.parse_app_logs]
type = "remap"
inputs = ["docker_logs"]
source = '''
  # Parse nested JSON in log field
  structured, err = parse_json(.message)
  if err == null {
    . = merge(., structured)
    del(.message)
  }

  # Add metadata
  .host = get_env_var!("HOSTNAME")
  .environment = get_env_var("ENVIRONMENT") ?? "production"

  # Normalize log level
  .level = downcase(string(.level) ?? "info")

  # Parse timestamp if present
  if exists(.timestamp) {
    .timestamp = parse_timestamp!(.timestamp, format: "%+")
  }
'''

# Filter out health check noise
[transforms.filter_noise]
type = "filter"
inputs = ["parse_app_logs"]
condition = '''
  !match(string(.path) ?? "", r'/health|/ping|/metrics')
'''

# Route logs by severity
[transforms.route_by_level]
type = "route"
inputs = ["filter_noise"]

[transforms.route_by_level.route]
errors = '.level == "error" || .level == "critical" || .level == "fatal"'
warnings = '.level == "warn" || .level == "warning"'
info = '.level == "info" || .level == "debug"'

# Enrich error logs with additional context
[transforms.enrich_errors]
type = "remap"
inputs = ["route_by_level.errors"]
source = '''
  .alert = true
  .requires_attention = true
'''

# ========================
# SINKS (Destinations)
# ========================

# All logs to Elasticsearch
[sinks.elasticsearch]
type = "elasticsearch"
inputs = ["filter_noise", "journald"]
endpoints = ["https://elasticsearch.yourcompany.internal:9200"]
auth.strategy = "basic"
auth.user = "vector"
auth.password = "${ES_PASSWORD}"
index = "logs-{{ .environment }}-{{ now() | strftime(\"%Y.%m.%d\") }}"
bulk.action = "create"
compression = "gzip"

[sinks.elasticsearch.buffer]
type = "disk"
max_size = 268435456  # 256MB
when_full = "block"

# Error logs to S3 for long-term retention
[sinks.s3_errors]
type = "aws_s3"
inputs = ["route_by_level.errors"]
bucket = "your-log-archive"
region = "us-east-1"
key_prefix = "errors/%Y/%m/%d/"
compression = "gzip"
encoding.codec = "json"

[sinks.s3_errors.auth]
access_key_id = "${AWS_ACCESS_KEY_ID}"
secret_access_key = "${AWS_SECRET_ACCESS_KEY}"

# Alert on critical errors via HTTP to Slack/PagerDuty webhook
[sinks.alert_errors]
type = "http"
inputs = ["enrich_errors"]
uri = "${SLACK_WEBHOOK_URL}"
method = "post"
encoding.codec = "json"
request.rate_limit_num = 10   # Max 10 alerts per second (de-duplicate)
request.rate_limit_duration_secs = 1

# Internal metrics (Vector's own performance)
[sources.vector_metrics]
type = "internal_metrics"

[sinks.prometheus_metrics]
type = "prometheus_exporter"
inputs = ["vector_metrics"]
address = "0.0.0.0:9598"
```

---

## Docker Compose Deployment

```yaml
# docker-compose.yml
version: "3.8"
services:
  vector:
    image: timberio/vector:latest-distroless-libc
    container_name: vector
    restart: unless-stopped
    volumes:
      - ./vector/vector.toml:/etc/vector/vector.toml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log:/var/log:ro
      - vector-data:/var/lib/vector
    environment:
      - ES_PASSWORD=${ES_PASSWORD}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - SLACK_WEBHOOK_URL=${SLACK_WEBHOOK_URL}
      - ENVIRONMENT=production
      - HOSTNAME=${HOSTNAME}
    ports:
      - "8686:8686"   # HTTP input
      - "9598:9598"   # Prometheus metrics
    # Needs Docker socket access
    group_add:
      - "999"

volumes:
  vector-data:
```

---

## Aggregator Config (Centralized)

The aggregator receives from all agent nodes:

```toml
# /etc/vector/aggregator.toml (on your central log server)

[sources.from_agents]
type = "vector"
address = "0.0.0.0:6000"
# Enable TLS for production:
# tls.enabled = true
# tls.cert_file = "/etc/vector/certs/server.crt"
# tls.key_file = "/etc/vector/certs/server.key"

[transforms.deduplicate]
type = "dedupe"
inputs = ["from_agents"]
fields.match = ["message", "host", "container_name"]
cache.num_events = 5000

[sinks.loki]
type = "loki"
inputs = ["deduplicate"]
endpoint = "http://loki.yourcompany.internal:3100"
encoding.codec = "json"

[sinks.loki.labels]
level = "{{ .level }}"
env = "{{ .environment }}"
service = "{{ .container_name }}"
host = "{{ .host }}"
```

On each agent, forward to aggregator:

```toml
[sinks.aggregator]
type = "vector"
inputs = ["parse_app_logs", "journald"]
address = "aggregator.yourcompany.internal:6000"
compression = true
```

---

## VRL (Vector Remap Language) Examples

VRL is Vector's transformation DSL. Common patterns:

```toml
# Extract fields from a structured log line
[transforms.parse_nginx_access]
type = "remap"
inputs = ["nginx_logs"]
source = '''
  . = parse_nginx_log!(string!(.message), format: "combined")
  .bytes_sent = to_int!(.size)
  .response_time_ms = to_float!(.request_time) * 1000
'''

# Mask PII before sending to log store
[transforms.mask_pii]
type = "remap"
inputs = ["app_logs"]
source = '''
  if exists(.email) {
    .email = redact(.email, filters: ["email"])
  }
  if exists(.credit_card) {
    .credit_card = redact(.credit_card, filters: ["credit_card_number"])
  }
  if exists(.ssn) {
    del(.ssn)
  }
'''

# Add request latency percentile tagging
[transforms.tag_latency]
type = "remap"
inputs = ["api_logs"]
source = '''
  latency_ms = to_float!(.response_time_ms)
  .latency_tier = if latency_ms < 100 {
    "fast"
  } else if latency_ms < 500 {
    "normal"
  } else if latency_ms < 2000 {
    "slow"
  } else {
    "critical"
  }
'''
```

---

## Monitoring Vector Itself

```bash
# Check Vector status
systemctl status vector

# Watch the API (enable api.enabled = true in config)
curl http://localhost:8686/health
curl http://localhost:8686/components

# Scrape Prometheus metrics
curl http://localhost:9598/metrics \
  | grep -E "vector_(events|errors|processed)"

# Real-time component stats
vector top  # requires vector CLI
```

---

## Related Reading

- [How to Set Up Fluentd for Log Collection](/remote-work-tools/how-to-set-up-fluentd-for-log-collection/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [Best Tools for Remote Team Error Tracking](/remote-work-tools/best-tools-remote-team-error-tracking/)

---

## Related Articles

- [How to Set Up Fluentd for Log Collection](/remote-work-tools/how-to-set-up-fluentd-for-log-collection/)
- [How to Write Async Daily Logs That Help Future Team Members](/remote-work-tools/how-to-write-async-daily-logs-that-help-future-team-members/)
- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)
- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [Optimize Docker for Slow Connections When Working Remotely](/remote-work-tools/docker-optimize-slow-connection-remote-work/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
