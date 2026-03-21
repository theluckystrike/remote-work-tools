---
layout: default
title: "Prometheus Monitoring Setup for Remote Infrastructure"
description: "Set up Prometheus and Grafana to monitor remote servers, containers, and services. Covers exporters, alerting rules, and dashboard config for distributed infra."
date: 2026-03-21
author: theluckystrike
permalink: /prometheus-monitoring-remote-infrastructure/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Remote infrastructure needs observability. Without it, you find out about a crashed service when a client emails you, not when it goes down at 3am. Prometheus scrapes metrics from your servers and containers every 15 seconds. Grafana turns those metrics into dashboards. Alertmanager sends you a page before the client notices.

This guide builds a complete monitoring stack: Prometheus, Grafana, and Node Exporter on a dedicated monitoring server, with targets across your fleet.

## Architecture

```
Your servers (targets)
  ├── app-server-1: node_exporter :9100
  ├── app-server-2: node_exporter :9100
  └── db-server: node_exporter :9100 + postgres_exporter :9187

Monitoring server
  ├── Prometheus :9090  (scrapes targets every 15s)
  ├── Grafana :3000     (queries Prometheus)
  └── Alertmanager :9093 (receives alerts, sends to Slack/PagerDuty)
```

## Install on the Monitoring Server

```bash
# docker-compose.monitoring.yml
version: "3.9"

volumes:
  prometheus_data: {}
  grafana_data: {}

services:
  prometheus:
    image: prom/prometheus:v2.51.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=90d"
      - "--web.enable-lifecycle"

  grafana:
    image: grafana/grafana:10.4.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=https://metrics.yourdomain.com

  alertmanager:
    image: prom/alertmanager:v0.27.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
```

## Prometheus Scrape Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    env: "production"

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets:
          - "app-server-1.internal:9100"
          - "app-server-2.internal:9100"
          - "db-server.internal:9100"
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: "([^:]+):.*"
        replacement: "$1"

  - job_name: "postgres"
    static_configs:
      - targets:
          - "db-server.internal:9187"
```

## Install Node Exporter on Each Target

```bash
# On each server you want to monitor
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvf node_exporter-1.7.0.linux-amd64.tar.gz
sudo mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude="^/(sys|proc|dev|host|etc)($$|/)" \
  --web.listen-address="0.0.0.0:9100"
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Create user
sudo useradd -rs /bin/false node_exporter

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

# Verify
curl http://localhost:9100/metrics | head -20
```

## Firewall Rules

Node exporter port 9100 should only be reachable from the monitoring server — not the public internet.

```bash
# On each target server (using ufw)
sudo ufw allow from MONITORING_SERVER_IP to any port 9100 proto tcp
sudo ufw deny 9100

# Verify
sudo ufw status | grep 9100
```

## Alerting Rules

```yaml
# rules/node.yml
groups:
  - name: node_alerts
    rules:
      - alert: HostDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Host {{ $labels.instance }} is unreachable"
          description: "Prometheus cannot scrape {{ $labels.instance }} for 1 minute"

      - alert: HighCPU
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% for 5 minutes"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanize }}% disk remaining on /"

      - alert: HighMemory
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}%"
```

## Alertmanager Config

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname", "instance"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "slack"
  routes:
    - match:
        severity: critical
      receiver: "pagerduty"
      continue: true

receivers:
  - name: "slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        channel: "#alerts"
        title: "{{ .GroupLabels.alertname }}"
        text: "{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}"
        send_resolved: true

  - name: "pagerduty"
    pagerduty_configs:
      - routing_key: "YOUR_PAGERDUTY_INTEGRATION_KEY"
        description: "{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}"

inhibit_rules:
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    equal: ["instance"]
```

The `inhibit_rules` block silences warning alerts when a critical alert is already firing for the same instance — so you get one alert, not five.

## Grafana Dashboard Provisioning

Instead of clicking through the GUI, provision dashboards as code:

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: Default
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

Import the Node Exporter Full dashboard (ID 1860) from grafana.com — it covers CPU, memory, disk, network, and load average in a single view without any manual panel configuration.

```bash
# Download and save to provisioning directory
curl -o grafana/provisioning/dashboards/node-exporter-full.json \
  "https://grafana.com/api/dashboards/1860/revisions/latest/download"
```

## Start the Stack

```bash
docker compose -f docker-compose.monitoring.yml up -d

# Check Prometheus targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'

# Should return "up" for each target
```

Access Grafana at port 3000, log in with admin / your GRAFANA_PASSWORD, and your Node Exporter dashboards appear automatically.

## Query Examples

```promql
# CPU usage per instance (last 5 minutes)
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Available disk space percentage
(node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Network receive rate (bytes/sec)
irate(node_network_receive_bytes_total{device!="lo"}[5m])

# Load average relative to CPU count
node_load1 / count by(instance) (node_cpu_seconds_total{mode="idle"})
```

## Related Reading

- [How to Secure Your Remote Team CI/CD Pipeline from Supply Chain Attacks](/remote-work-tools/how-to-secure-remote-team-ci-cd-pipeline-from-supply-chain-a/)
- [Home Lab Setup Guide for Remote Developers](/remote-work-tools/home-lab-setup-guide-remote-developers/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
