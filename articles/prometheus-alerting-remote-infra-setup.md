---
layout: default
title: "Prometheus Alerting for Remote Infrastructure"
description: "Configure Prometheus alerting rules, Alertmanager routing, and PagerDuty/Slack integrations to catch remote infra failures before users notice"
date: 2026-03-22
author: theluckystrike
permalink: /prometheus-alerting-remote-infra-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## Prometheus Alerting for Remote Infrastructure

Running distributed infrastructure across cloud regions means failures happen at odd hours, often when no one is watching. Prometheus alerting combined with Alertmanager routes the right signal to the right person without waking everyone for a disk filling on a dev box.

This guide covers writing alert rules that fire on real conditions, routing alerts through Alertmanager, and integrating with Slack and PagerDuty.

---

## Prerequisites

- Prometheus already scraping targets (see the monitoring setup guide)
- Alertmanager installed alongside Prometheus
- A Slack webhook URL or PagerDuty integration key

---

## Writing Alert Rules

Alert rules live in separate `.rules.yml` files and are loaded by Prometheus via `rule_files` in `prometheus.yml`.

**`/etc/prometheus/rules/infra.rules.yml`**

```yaml
groups:
  - name: infra
    interval: 30s
    rules:

      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
          team: ops
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for 2 minutes."

      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 5m
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "CPU usage above 90% on {{ $labels.instance }}"
          description: "CPU is at {{ $value | printf \"%.1f\" }}% for 5 minutes."

      - alert: DiskAlmostFull
        expr: (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} / node_filesystem_size_bytes) * 100 < 10
        for: 10m
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "Disk < 10% free on {{ $labels.instance }}"
          description: "Filesystem {{ $labels.mountpoint }} has {{ $value | printf \"%.1f\" }}% free."

      - alert: MemoryPressure
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
          team: ops
        annotations:
          summary: "Memory critical on {{ $labels.instance }}"
          description: "Available memory is {{ $value | printf \"%.1f\" }}%."

      - alert: ContainerRestarting
        expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 5
        for: 0m
        labels:
          severity: warning
          team: dev
        annotations:
          summary: "Container {{ $labels.container }} is restarting"
          description: "Container in pod {{ $labels.pod }} has restarted {{ $value }} times in 15 minutes."
```

Reference this file in `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.rules.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

Reload Prometheus after changes:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## Installing Alertmanager

```bash
# Download and install
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64/alertmanager /usr/local/bin/
sudo mv alertmanager-0.27.0.linux-amd64/amtool /usr/local/bin/

# Create config directory
sudo mkdir -p /etc/alertmanager
```

---

## Alertmanager Configuration

Alertmanager routes alerts based on labels, groups related alerts, and deduplicates. A well-designed routing tree prevents notification storms.

**`/etc/alertmanager/alertmanager.yml`**

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
  pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

templates:
  - /etc/alertmanager/templates/*.tmpl

route:
  group_by: ["alertname", "instance"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: slack-ops

  routes:
    # Critical alerts go to PagerDuty
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true  # also send to slack

    # Dev team alerts go to dev channel
    - match:
        team: dev
      receiver: slack-dev

    # Business hours only for warnings
    - match:
        severity: warning
      receiver: slack-ops
      mute_time_intervals:
        - outside-business-hours

receivers:
  - name: slack-ops
    slack_configs:
      - channel: "#ops-alerts"
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        color: '{{ if eq .Status "firing" }}{{ if eq .CommonLabels.severity "critical" }}danger{{ else }}warning{{ end }}{{ else }}good{{ end }}'

  - name: slack-dev
    slack_configs:
      - channel: "#dev-alerts"
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ "\n" }}{{ end }}'

  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: "YOUR_PAGERDUTY_INTEGRATION_KEY"
        description: '{{ template "pagerduty.description" . }}'
        severity: '{{ .CommonLabels.severity }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          instance: '{{ .CommonLabels.instance }}'

inhibit_rules:
  # Suppress warnings if a critical alert is already firing for the same instance
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["instance"]

time_intervals:
  - name: outside-business-hours
    time_intervals:
      - times:
          - start_time: "00:00"
            end_time: "09:00"
          - start_time: "17:00"
            end_time: "24:00"
        weekdays: ["monday:friday"]
      - weekdays: ["saturday", "sunday"]
```

---

## Slack Message Templates

**`/etc/alertmanager/templates/slack.tmpl`**

```
{{ define "slack.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }}
*Severity:* {{ .Labels.severity }}
*Instance:* {{ .Labels.instance }}
*Details:* {{ .Annotations.description }}
*Started:* {{ .StartsAt | since }}
{{ end }}
{{ end }}
```

---

## Systemd Service for Alertmanager

**`/etc/systemd/system/alertmanager.service`**

```ini
[Unit]
Description=Alertmanager
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=:9093 \
  --log.level=info
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

---

## Testing Alerts

Use `amtool` to validate config and fire test alerts:

```bash
# Validate config syntax
amtool check-config /etc/alertmanager/alertmanager.yml

# List active alerts
amtool alert query --alertmanager.url=http://localhost:9093

# Silence an alert during maintenance
amtool silence add \
  --alertmanager.url=http://localhost:9093 \
  --duration=2h \
  --comment="Planned maintenance" \
  alertname=InstanceDown instance=web-01:9100

# Test alert delivery with curl
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {"alertname": "TestAlert", "severity": "warning", "instance": "test-host"},
    "annotations": {"summary": "Test alert", "description": "This is a test."},
    "startsAt": "2026-03-22T00:00:00Z"
  }]'
```

---

## Alert Rule Best Practices for Remote Teams

**Use `for` duration wisely**: A `for: 0m` fires immediately; use it only for alerts that need instant action (pod crash loops). For infrastructure metrics, 2-5 minutes prevents false fires from scrape gaps.

**Label everything**: Add `env`, `region`, and `team` labels to your scrape targets so alerts route correctly without extra config. Set labels in Prometheus scrape configs:

```yaml
scrape_configs:
  - job_name: web-prod
    static_configs:
      - targets: ["web-01:9100", "web-02:9100"]
        labels:
          env: production
          region: us-east-1
          team: ops
```

**Inhibit noisy derived alerts**: If a host goes down, suppress all the application-level alerts from that host using inhibit rules keyed on `instance`.

**Review alert fatigue weekly**: If a channel gets more than 20 alerts per week, either the threshold is wrong or the underlying problem needs fixing. Track alert volume in Grafana with:

```
count_over_time(ALERTS{alertstate="firing"}[7d])
```

---

## Related Articles

- [Prometheus Monitoring Setup for Remote Infrastructure](/remote-work-tools/prometheus-monitoring-remote-infrastructure/)
- [Best Deploy Workflow for a Remote Infrastructure Team of 3](/remote-work-tools/best-deploy-workflow-for-a-remote-infrastructure-team-of-3/)
- [Best Practice for Remote Team README Files in Repositories](/remote-work-tools/best-practice-for-remote-team-readme-files-in-repositories-s/)
- [Remote Team Charter Template Guide 2026](/remote-work-tools/remote-team-charter-template-guide-2026/)
- [Best Tools for Remote Team Daily Health Checks](/remote-work-tools/best-tools-remote-team-daily-health-checks/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
