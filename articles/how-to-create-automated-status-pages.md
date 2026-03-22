---
layout: default
title: "How to Create Automated Status Pages"
description: "Build automated status pages with Upptime, Freshping, or Gatus that update on every incident without manual intervention from your team"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-automated-status-pages/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Manual status pages are a lie — someone always forgets to update them during an incident. Automated status pages read actual service health in real time, post incidents without human intervention, and close them when monitors recover. For remote teams, they're also the first thing customers check before filing a support ticket.

---

## Upptime: GitHub-Powered Status Page

Upptime runs entirely within GitHub Actions and GitHub Pages. Monitors are defined in a YAML file; GitHub Actions runs checks every 5 minutes and commits results to the repo. Zero infrastructure to manage.

```bash
# Use the Upptime template
gh repo create your-org/status --template upptime/upptime --public
cd status
```

Configure monitors in `.upptimerc.yml`:

```yaml
owner: your-org
repo: status
userAgent: your-org/status
sites:
  - name: Main Website
    url: https://yourcompany.com
    expectedStatusCodes: [200, 302]

  - name: API
    url: https://api.yourcompany.com/health
    expectedStatusCodes: [200]
    timeout: 10000

  - name: Dashboard
    url: https://app.yourcompany.com/login
    expectedStatusCodes: [200]

  - name: Documentation
    url: https://docs.yourcompany.com
    expectedStatusCodes: [200]

  - name: Webhooks
    url: https://webhooks.yourcompany.com/ping
    method: POST
    body: '{"ping": true}'
    headers:
      Content-Type: application/json
    expectedStatusCodes: [200]

assignees:
  - your-github-username

status-website:
  baseUrl: https://status.yourcompany.com
  name: YourCompany Status
  introTitle: Service Status
  introMessage: Real-time status for YourCompany services
  navbar:
    - title: Support
      href: https://support.yourcompany.com
```

Add secrets to the repo:

```
GH_PAT          = GitHub personal access token (repo + workflow scopes)
```

Enable GitHub Pages on the `gh-pages` branch. Upptime's Actions workflow will automatically:
- Check each URL every 5 minutes
- Open GitHub Issues when monitors go down
- Close Issues when monitors recover
- Update `README.md` with current uptime badges
- Publish the status page to GitHub Pages

---

## Gatus: Self-Hosted, Config-Driven

Gatus is a self-hosted status page and uptime monitor with more flexibility than Upptime. It supports HTTP, TCP, and DNS checks with custom conditions.

Docker Compose deployment:

```yaml
# docker-compose.yml
version: "3.8"
services:
  gatus:
    image: twinproduction/gatus:latest
    container_name: gatus
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./config/gatus.yml:/config/config.yaml:ro
      - gatus-data:/data

volumes:
  gatus-data:
```

Full configuration at `config/gatus.yml`:

```yaml
# config/gatus.yml
web:
  port: 8080

storage:
  type: sqlite
  path: /data/gatus.db

alerting:
  slack:
    webhook-url: "https://hooks.slack.com/services/T.../B.../..."
    default-alert:
      enabled: true
      failure-threshold: 2
      success-threshold: 1
      description: "Status change detected"

endpoints:
  - name: API Gateway
    group: core
    url: https://api.yourcompany.com/health
    interval: 30s
    conditions:
      - "[STATUS] == 200"
      - "[RESPONSE_TIME] < 500"
      - "[BODY].status == healthy"
    alerts:
      - type: slack
        failure-threshold: 3
        description: "API Gateway is down"

  - name: Database Connectivity
    group: core
    url: tcp://db.yourcompany.internal:5432
    interval: 60s
    conditions:
      - "[CONNECTED] == true"
    alerts:
      - type: slack

  - name: Payment Processor
    group: integrations
    url: https://api.yourcompany.com/health/stripe
    interval: 60s
    conditions:
      - "[STATUS] == 200"
      - "[BODY].stripe == reachable"
    alerts:
      - type: slack
        failure-threshold: 1
        description: "Stripe integration unhealthy"

  - name: DNS Resolution
    group: infrastructure
    url: "8.8.8.8"
    dns:
      query-name: api.yourcompany.com
      query-type: A
    interval: 120s
    conditions:
      - "[DNS_RCODE] == NOERROR"

  - name: SSL Certificate
    url: https://yourcompany.com
    interval: 3600s
    conditions:
      - "[STATUS] == 200"
      - "[CERTIFICATE_EXPIRATION] > 336h"  # 14 days
    alerts:
      - type: slack
        failure-threshold: 1
        description: "SSL cert expiring soon"
```

Expose via Nginx at `status.yourcompany.com`:

```nginx
server {
    listen 443 ssl http2;
    server_name status.yourcompany.com;

    ssl_certificate     /etc/letsencrypt/live/status.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/status.yourcompany.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
}
```

---

## Freshping (SaaS Option)

Freshping offers a generous free tier (50 checks, 1-minute intervals) with zero infrastructure overhead. Use it when you need something running in 10 minutes:

1. Sign up at freshping.io
2. Add checks (URL, keyword, SSL, ping)
3. Create a status page under **Status Pages > New**
4. Add a custom domain via CNAME: `status.yourcompany.com` → `yourstatusdomain.freshping.io`

For API-driven check management:

```bash
# Create a check via API
curl -X POST \
  -H "Authorization: Bearer $FRESHPING_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "application_id": 1,
    "name": "API Health",
    "request_url": "https://api.yourcompany.com/health",
    "request_type": "GET",
    "check_frequency": 60,
    "contact_groups": [1]
  }' \
  "https://api.freshping.io/v1/checks/"
```

---

## Incident Automation with PagerDuty

Wire your status page to PagerDuty so incidents trigger on-call pages automatically:

```bash
# Gatus PagerDuty alerting
# Add to gatus.yml alerting section
alerting:
  pagerduty:
    integration-key: "your_pagerduty_integration_key"
    default-alert:
      enabled: true
      failure-threshold: 2
      success-threshold: 1
```

For Upptime, add a GitHub Actions step that creates PagerDuty incidents:

```yaml
# .github/workflows/uptime.yml (add to existing Upptime workflow)
- name: Create PagerDuty incident
  if: failure()
  run: |
    curl -X POST \
      -H "Content-Type: application/json" \
      -d '{
        "routing_key": "${{ secrets.PAGERDUTY_INTEGRATION_KEY }}",
        "event_action": "trigger",
        "payload": {
          "summary": "Status check failed: ${{ matrix.sites.name }}",
          "severity": "critical",
          "source": "upptime"
        }
      }' \
      https://events.pagerduty.com/v2/enqueue
```

---

## Custom Maintenance Windows

Schedule planned maintenance so the status page shows "Scheduled Maintenance" instead of "Down":

For Gatus, use external endpoint toggling:

```bash
#!/bin/bash
# scripts/maintenance.sh
# Usage: ./maintenance.sh start|end "Reason text"

ACTION=$1
REASON=$2
GATUS_URL="https://status.yourcompany.com"

if [ "$ACTION" = "start" ]; then
  # Post to internal Slack channel
  curl -s -X POST \
    -H 'Content-type: application/json' \
    --data "{\"text\":\"Maintenance started: $REASON\"}" \
    "$SLACK_WEBHOOK"

  # Update your status page banner via API or config reload
  docker exec gatus sh -c "echo 'MAINTENANCE_MODE=true' > /tmp/maintenance"
  docker restart gatus
fi
```

For Upptime, create a GitHub Issue tagged `maintenance` and the status page will show it as scheduled maintenance automatically.

---

## Status Badge for Your README

Add live status badges to service READMEs:

```markdown
<!-- Upptime badge -->
[![API Status](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/your-org/status/master/api/api/uptime.json)](https://status.yourcompany.com)

<!-- Gatus badge (custom endpoint) -->
[![API Status](https://status.yourcompany.com/api/v1/endpoints/core_api-gateway/health/badge.svg)](https://status.yourcompany.com)
```

---

## Related Reading

- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Automate SSL Certificate Renewal](/remote-work-tools/how-to-automate-ssl-certificate-renewal/)
- [How to Automate Infrastructure Cost Alerts](/remote-work-tools/how-to-automate-infrastructure-cost-alerts/)

---

## Related Articles

- [How to Write Async Status Updates That Managers Actually](/remote-work-tools/how-to-write-async-status-updates-that-managers-actually-read/)
- [AI Project Status Generator for Remote Teams Pulling](/remote-work-tools/ai-project-status-generator-for-remote-teams-pulling-data-fr/)
- [GitHub Actions Workflow for Remote Dev Teams](/remote-work-tools/github-actions-remote-dev-workflow/)
- [Client Project Status Dashboard Setup for Remote Agency](/remote-work-tools/client-project-status-dashboard-setup-for-remote-agency-team/)
- [Best Format for Remote Team Weekly Written Status Update](/remote-work-tools/best-format-for-remote-team-weekly-written-status-update-rep/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
