---
layout: default
title: "How to Create Automated API Health Dashboards"
description: "Build automated API health dashboards with Prometheus, Grafana, and synthetic monitors so remote teams see outages before users do"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-create-automated-api-health-dashboards/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, api]
---

{% raw %}

When your team is distributed across time zones, nobody wants to wake up to a midnight Slack storm about an API that went down six hours ago. This guide builds an automated API health dashboard that fires alerts the moment something degrades. not when a user reports it.

Stack Overview

- Prometheus. metrics scrape and storage
- Grafana. dashboards and alert routing
- Blackbox Exporter. HTTP/TCP synthetic probing
- Alertmanager. PagerDuty and Slack routing
- k6. scheduled load-based health checks

Docker Compose Setup

```yaml
docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.51.0
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    restart: unless-stopped

  blackbox:
    image: prom/blackbox-exporter:v0.25.0
    volumes:
      - ./blackbox:/etc/blackbox_exporter
    ports:
      - "9115:9115"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.4.1
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=${SMTP_HOST}
    ports:
      - "3000:3000"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.27.0
    volumes:
      - ./alertmanager:/etc/alertmanager
    ports:
      - "9093:9093"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

Prometheus Configuration

```yaml
prometheus/prometheus.yml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "rules/*.yml"

scrape_configs:
  # Blackbox HTTP probing for each API endpoint
  - job_name: 'api-health'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://api.example.com/health
          - https://api.example.com/v1/users
          - https://api.example.com/v1/products
          - https://payments.example.com/health
          - https://auth.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115

  # Latency percentile probe (POST endpoint)
  - job_name: 'api-post-health'
    metrics_path: /probe
    params:
      module: [http_post_2xx]
    static_configs:
      - targets:
          - https://api.example.com/v1/orders
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115

  # SSL cert expiry monitoring
  - job_name: 'ssl-expiry'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - api.example.com:443
          - payments.example.com:443
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

Blackbox Modules

```yaml
blackbox/config.yml
modules:
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 202, 204]
      method: GET
      follow_redirects: true
      fail_if_ssl: false
      fail_if_not_ssl: true
      tls_config:
        insecure_skip_verify: false
      headers:
        Accept: application/json
        Authorization: Bearer ${API_HEALTH_TOKEN}

  http_post_2xx:
    prober: http
    timeout: 15s
    http:
      valid_status_codes: [200, 201]
      method: POST
      headers:
        Content-Type: application/json
        Authorization: Bearer ${API_HEALTH_TOKEN}
      body: '{"probe": true}'

  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      tls: true
```

Alert Rules

```yaml
prometheus/rules/api.yml
groups:
  - name: api_availability
    interval: 30s
    rules:
      - alert: APIEndpointDown
        expr: probe_success{job="api-health"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "API endpoint down: {{ $labels.instance }}"
          description: "{{ $labels.instance }} has been down for more than 1 minute"
          runbook: "https://wiki.example.com/runbooks/api-down"

      - alert: APISlowResponse
        expr: probe_duration_seconds{job="api-health"} > 2
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Slow API response: {{ $labels.instance }}"
          description: "{{ $labels.instance }} responding in {{ $value | humanizeDuration }}"

      - alert: APIHighErrorRate
        expr: |
          (
            sum(rate(probe_http_status_code{job="api-health",status_code!~"2.."}[5m]))
            /
            sum(rate(probe_http_status_code{job="api-health"}[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "API error rate above 5%"
          description: "Error rate: {{ $value | humanizePercentage }}"

      - alert: SSLCertExpiringSoon
        expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 14
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL cert expiring: {{ $labels.instance }}"
          description: "Cert expires in {{ $value | humanizeDuration }}"
```

Alertmanager Routing

```yaml
alertmanager/alertmanager.yml
global:
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-warnings'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    - match:
        severity: critical
      receiver: 'slack-critical'

receivers:
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#api-health'
        title: '{{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Instance:* {{ .Labels.instance }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook }}
          {{ end }}
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#incidents'
        title: 'CRITICAL: {{ .GroupLabels.alertname }}'
        color: 'danger'
        text: |
          {{ range .Alerts }}*{{ .Annotations.summary }}*
          {{ .Annotations.description }}
          {{ end }}
        send_resolved: true

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_ROUTING_KEY}'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        severity: critical

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance']
```

Grafana Dashboard JSON (Key Panels)

```json
{
  "panels": [
    {
      "title": "API Availability (24h)",
      "type": "stat",
      "targets": [{
        "expr": "avg_over_time(probe_success{job=\"api-health\"}[24h]) * 100",
        "legendFormat": "{{instance}}"
      }],
      "thresholds": {
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 99, "color": "yellow"},
          {"value": 99.9, "color": "green"}
        ]
      }
    },
    {
      "title": "Response Time P95",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(probe_duration_seconds_bucket{job=\"api-health\"}[5m]))",
        "legendFormat": "p95 {{instance}}"
      }]
    },
    {
      "title": "SSL Cert Days Remaining",
      "type": "gauge",
      "targets": [{
        "expr": "(probe_ssl_earliest_cert_expiry{job=\"ssl-expiry\"} - time()) / 86400",
        "legendFormat": "{{instance}}"
      }],
      "thresholds": {
        "steps": [
          {"value": 0, "color": "red"},
          {"value": 14, "color": "yellow"},
          {"value": 30, "color": "green"}
        ]
      }
    }
  ]
}
```

Provision Dashboard Automatically

```yaml
grafana/provisioning/dashboards/default.yml
apiVersion: 1
providers:
  - name: 'API Health'
    orgId: 1
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

k6 Scheduled Health Script

```javascript
// health-check.js. run via cron every 5 minutes
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 1,
  iterations: 1,
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.API_BASE_URL || 'https://api.example.com';
const TOKEN = __ENV.API_TOKEN;

const headers = { Authorization: `Bearer ${TOKEN}`, 'Content-Type': 'application/json' };

export default function () {
  const endpoints = [
    { method: 'GET', path: '/health' },
    { method: 'GET', path: '/v1/users?limit=1' },
    { method: 'GET', path: '/v1/products?limit=1' },
  ];

  for (const ep of endpoints) {
    const res = http.request(ep.method, `${BASE_URL}${ep.path}`, null, { headers });
    check(res, {
      [`${ep.path} status 2xx`]: (r) => r.status >= 200 && r.status < 300,
      [`${ep.path} < 1s`]: (r) => r.timings.duration < 1000,
    });
  }
}
```

```bash
cron: */5 * * * * /usr/local/bin/k6 run \
  -e API_BASE_URL=https://api.example.com \
  -e API_TOKEN=$(cat /run/secrets/api-token) \
  /opt/health-checks/health-check.js \
  --out influxdb=http://influxdb:8086/k6 2>&1 | logger -t k6-health
```

Related Reading

- [How to Set Up Caddy for Internal Tools](/how-to-set-up-caddy-for-internal-tools/)
- [How to Create Automated Database Indexing Alerts](/how-to-create-automated-database-indexing-alerts/)
- [How to Create Automated Performance Budgets](/how-to-create-automated-performance-budgets/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
