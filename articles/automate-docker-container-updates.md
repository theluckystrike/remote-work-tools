---
layout: default
title: "How to Automate Docker Container Updates"
description: "Use Watchtower, Diun, and shell scripts to automate Docker container updates with rollback safety, Slack notifications, and minimal downtime for remote infra"
date: 2026-03-22
author: theluckystrike
permalink: /automate-docker-container-updates/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Automate Docker Container Updates

Manually SSHing into servers to run `docker pull && docker restart` is a maintenance tax that compounds across a distributed team. Automating container updates with proper health checks and rollback paths means new images ship without anyone touching a terminal.

For remote teams, manual update procedures are especially costly. An update that should take 5 minutes becomes a 30-minute coordination task when the engineer who owns the server is asleep and the engineer who needs the fix is wide awake. Automated updates with notifications solve the coordination problem: images deploy, Slack confirms, and no one needs to wake anyone up.

---

## Approach 1: Watchtower (Pull-Based Auto-Update)

Watchtower watches running containers, polls their registries, and restarts them when new images appear. It's the simplest path for non-orchestrated Docker hosts.

**Run Watchtower alongside your stack:**

```yaml
# docker-compose.yml
version: "3.8"
services:
  myapp:
    image: ghcr.io/yourorg/myapp:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    labels:
      - "com.centurylinklabs.watchtower.enable=true"

  watchtower:
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ~/.docker/config.json:/config.json:ro  # registry auth
    environment:
      # Check every 300 seconds
      WATCHTOWER_POLL_INTERVAL: "300"
      # Only update containers with the label above
      WATCHTOWER_LABEL_ENABLE: "true"
      # Wait for containers to fully start before considering healthy
      WATCHTOWER_ROLLING_RESTART: "true"
      # Send Slack notification on updates
      WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL: "https://hooks.slack.com/services/YOUR/HOOK"
      WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER: "watchtower-prod"
      WATCHTOWER_NOTIFICATIONS: "slack"
      # Clean up old images after update
      WATCHTOWER_CLEANUP: "true"
      # HTTP API for manual triggers
      WATCHTOWER_HTTP_API_UPDATE: "true"
      WATCHTOWER_HTTP_API_TOKEN: "your-secret-token"
    ports:
      - "8080:8080"
```

**Trigger an immediate update via API (useful from CI/CD):**

```bash
curl -H "Authorization: Bearer your-secret-token" \
  http://your-host:8080/v1/update
```

**Pin a container to prevent auto-update:**

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

Watchtower's label-based opt-in (`WATCHTOWER_LABEL_ENABLE: "true"`) is important in mixed environments. Databases and stateful services should never auto-update without explicit human approval. Apply the enable label only to stateless application containers.

---

## Approach 2: Diun (Notification Only, Manual Pull)

Diun watches registries and sends notifications when new images are available, without auto-updating. Use this when you want human approval before deploying.

```yaml
# diun/docker-compose.yml
version: "3.8"
services:
  diun:
    image: crazymax/diun:latest
    restart: unless-stopped
    volumes:
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: UTC
      LOG_LEVEL: info
      DIUN_WATCH_WORKERS: "20"
      DIUN_WATCH_SCHEDULE: "0 */6 * * *"   # every 6 hours
      DIUN_WATCH_JITTER: "30s"
      # Slack notifications
      DIUN_NOTIF_SLACK_WEBHOOKURL: "https://hooks.slack.com/services/YOUR/HOOK"
      DIUN_NOTIF_SLACK_CHANNEL: "#infra-updates"
      # Watch all running containers
      DIUN_PROVIDERS_DOCKER: "true"
      DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT: "true"
```

Diun posts to Slack when `ghcr.io/yourorg/myapp:latest` gets a new digest. Your team can then decide when to pull.

For regulated environments or services with strict change management requirements, Diun-plus-manual-approval is the correct model. The notification lands in `#infra-updates`, an engineer reviews the changelog and schedules the update during a maintenance window, and the actual deployment remains under explicit human control.

---

## Approach 3: Shell Script with Health Check and Rollback

For teams that want full control without external tools:

```bash
#!/bin/bash
# update-container.sh — safe in-place update with rollback
set -euo pipefail

SERVICE_NAME="${1:?Usage: $0 <service-name>}"
COMPOSE_FILE="${2:-/opt/myapp/docker-compose.yml}"
SLACK_HOOK="${SLACK_WEBHOOK_URL:-}"
LOG_FILE="/var/log/container-updates.log"

log() {
  echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $*" | tee -a "$LOG_FILE"
}

notify_slack() {
  local msg="$1"
  [[ -z "$SLACK_HOOK" ]] && return
  curl -s -X POST "$SLACK_HOOK" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"$msg\"}"
}

# Record current image digest before updating
CURRENT_IMAGE=$(docker-compose -f "$COMPOSE_FILE" config | grep "image:" | grep "$SERVICE_NAME" | awk '{print $2}')
CURRENT_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "$CURRENT_IMAGE" 2>/dev/null || echo "unknown")

log "Starting update for $SERVICE_NAME (current: $CURRENT_DIGEST)"

# Pull new image
if ! docker-compose -f "$COMPOSE_FILE" pull "$SERVICE_NAME"; then
  log "ERROR: Pull failed for $SERVICE_NAME"
  notify_slack ":x: Pull failed for \`$SERVICE_NAME\` on $(hostname)"
  exit 1
fi

NEW_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "$CURRENT_IMAGE" 2>/dev/null || echo "unknown")

if [[ "$CURRENT_DIGEST" == "$NEW_DIGEST" ]]; then
  log "No update available for $SERVICE_NAME"
  exit 0
fi

log "New image available: $NEW_DIGEST — restarting"

# Restart with health check
docker-compose -f "$COMPOSE_FILE" up -d --no-deps "$SERVICE_NAME"

# Wait up to 60 seconds for health check to pass
TIMEOUT=60
ELAPSED=0
while [[ $ELAPSED -lt $TIMEOUT ]]; do
  STATUS=$(docker-compose -f "$COMPOSE_FILE" ps -q "$SERVICE_NAME" | xargs docker inspect --format='{{.State.Health.Status}}' 2>/dev/null || echo "none")
  if [[ "$STATUS" == "healthy" || "$STATUS" == "none" ]]; then
    log "Container $SERVICE_NAME is up (status: $STATUS)"
    notify_slack ":white_check_mark: Updated \`$SERVICE_NAME\` on $(hostname) → ${NEW_DIGEST:0:20}"
    exit 0
  fi
  sleep 5
  ELAPSED=$((ELAPSED + 5))
done

# Health check timed out — rollback
log "ERROR: Health check timed out for $SERVICE_NAME — rolling back"
notify_slack ":warning: Update failed for \`$SERVICE_NAME\` — rolling back"

docker tag "$CURRENT_IMAGE" "${CURRENT_IMAGE}-rollback"  || true
docker-compose -f "$COMPOSE_FILE" up -d --no-deps --force-recreate "$SERVICE_NAME"
exit 1
```

**Cron job to run nightly at 2 AM:**

```bash
# /etc/cron.d/container-updates
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/HOOK

0 2 * * * root /usr/local/bin/update-container.sh myapp /opt/myapp/docker-compose.yml >> /var/log/container-updates.log 2>&1
```

The 2 AM UTC scheduling works well for teams with North America and Europe coverage — it's overnight for both regions. For teams with Asia-Pacific engineers who need updates during their workday, adjust to a time that avoids everyone's peak hours.

---

## Approach 4: GitHub Actions Webhook Trigger

Push a new image from CI/CD and have the production host pull immediately:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        run: |
          docker build -t ghcr.io/${{ github.repository }}:latest .
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:latest

      - name: Trigger Watchtower update
        run: |
          curl -H "Authorization: Bearer ${{ secrets.WATCHTOWER_TOKEN }}" \
            https://prod.yourcompany.com:8080/v1/update
```

---

## Healthcheck Configuration in Docker Images

The shell script rollback approach depends on containers having a proper `HEALTHCHECK` in their Dockerfile. Without it, Docker always reports status as `none` and the script can't detect a failed update:

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY . .
RUN npm ci --only=production

# Application must respond 200 on /health within 5 seconds
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "server.js"]
```

For Go services:

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server ./cmd/server

FROM alpine:3.19
RUN apk add --no-cache curl
COPY --from=builder /app/server /server

HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["/server"]
```

The `--start-period` parameter is critical: it gives the container time to initialize before health checks begin counting failures. Set it to your typical startup time plus a safety margin. An app that takes 15 seconds to warm up should have `--start-period=30s` to avoid false rollbacks immediately after a good deployment.

---

## Registry Authentication

For private registries, configure Docker credential helpers before running any update tooling:

```bash
# For GHCR
echo "$GITHUB_TOKEN" | docker login ghcr.io -u youruser --password-stdin

# For AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# The resulting ~/.docker/config.json is mounted into Watchtower
```

For ECR specifically, credentials expire every 12 hours. Run the login command on a cron schedule before Watchtower's poll interval:

```bash
# /etc/cron.d/ecr-auth — refresh ECR credentials every 6 hours
0 */6 * * * root aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com >> /var/log/ecr-auth.log 2>&1
```

---

## Keeping a Changelog of Updates

```bash
# Append to a simple update log that the team can review
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) | $SERVICE_NAME | $CURRENT_DIGEST -> $NEW_DIGEST" \
  >> /var/log/image-changelog.log

# Query last 20 updates
tail -20 /var/log/image-changelog.log
```

Post the changelog to Slack weekly so the team has visibility into what changed without checking server logs manually:

```bash
#!/bin/bash
# Weekly image update summary
UPDATES=$(grep "$(date -d '7 days ago' +%Y-%m)" /var/log/image-changelog.log | tail -20)
if [[ -n "$UPDATES" ]]; then
  curl -s -X POST "$SLACK_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"Weekly container update log:\n\`\`\`$UPDATES\`\`\`\"}"
fi
```

---

## Choosing the Right Approach

Each approach in this guide has a different risk profile and automation level:

| Approach | Automation Level | Human Approval | Best For |
|----------|-----------------|----------------|----------|
| Watchtower | Fully automatic | None | Staging environments, non-critical services |
| Diun | Notification only | Required | Production, regulated services |
| Shell script | Configurable (cron) | Optional | Teams wanting full control and custom logic |
| CI/CD webhook | Triggered by code merge | Via PR process | Teams already using GitHub Actions or Drone |

For most remote teams, a two-tier approach works well: Watchtower handles staging automatically so engineers always have a fresh environment to test against, while production uses the CI/CD webhook approach that requires a merge to `main` to trigger a deployment. This preserves the PR review process as the approval gate for production changes without adding a separate manual deployment step.

---

## Related Articles

- [Optimize Docker for Slow Connections When Working Remotely](/remote-work-tools/docker-optimize-slow-connection-remote-work/)
- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [How to Set Up Traefik Reverse Proxy](/remote-work-tools/how-to-set-up-traefik-reverse-proxy/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
