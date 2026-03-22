---
layout: default
title: "How to Set Up Gitea for Self-Hosted Git"
description: "Deploy Gitea with Docker for a lightweight self-hosted GitHub alternative with SSH, webhooks, Actions CI, and team access controls"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-gitea-self-hosted-git/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Gitea is a 60MB binary that gives your team GitHub-like features: repos, issues, pull requests, webhooks, and Gitea Actions (compatible with GitHub Actions syntax). Run it on a $6/month VPS and own your code. This guide covers a production Docker deployment with SSH, SMTP, and backup.

## Key Takeaways

- **Run it on a**: $6/month VPS and own your code.
- **Gitea is a 60MB**: binary that gives your team GitHub-like features: repos, issues, pull requests, webhooks, and Gitea Actions (compatible with GitHub Actions syntax).
- **Topics covered**: docker compose deployment, nginx reverse proxy, ssh configuration for team members
- **Practical guidance included**: Step-by-step setup and configuration instructions

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Docker Compose Deployment

```yaml
# docker-compose.yml
version: "3.8"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:1.21.4
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=${DB_PASSWORD}
      - GITEA__server__DOMAIN=git.example.com
      - GITEA__server__SSH_DOMAIN=git.example.com
      - GITEA__server__ROOT_URL=https://git.example.com/
      - GITEA__server__SSH_PORT=2222
      - GITEA__server__SSH_LISTEN_PORT=22
      - GITEA__mailer__ENABLED=true
      - GITEA__mailer__FROM=git@example.com
      - GITEA__mailer__PROTOCOL=smtp+startls
      - GITEA__mailer__SMTP_ADDR=${SMTP_HOST}
      - GITEA__mailer__SMTP_PORT=587
      - GITEA__mailer__USER=${SMTP_USER}
      - GITEA__mailer__PASSWD=${SMTP_PASSWORD}
      - GITEA__service__DISABLE_REGISTRATION=true
      - GITEA__service__REQUIRE_SIGNIN_VIEW=true
    restart: unless-stopped
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=gitea
    networks:
      - gitea
    volumes:
      - ./postgres:/var/lib/postgresql/data
```

```bash
# .env
DB_PASSWORD=strong-postgres-password
SMTP_HOST=smtp.sendgrid.net
SMTP_USER=apikey
SMTP_PASSWORD=your-sendgrid-key
```

```bash
# Start Gitea
docker compose up -d

# Check logs
docker compose logs -f server

# First run: visit http://server:3000 to complete setup wizard
# Or configure everything via docker-compose env vars (recommended)
```

### Step 2: Nginx Reverse Proxy

```nginx
# /etc/nginx/sites-available/gitea
server {
    listen 80;
    server_name git.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name git.example.com;

    ssl_certificate /etc/letsencrypt/live/git.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/git.example.com/privkey.pem;

    client_max_body_size 512m;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo certbot --nginx -d git.example.com
sudo nginx -t && sudo systemctl reload nginx
```

### Step 3: SSH Configuration for Team Members

```bash
# On your local machine, add to ~/.ssh/config
Host gitea
  HostName git.example.com
  User git
  Port 2222
  IdentityFile ~/.ssh/id_ed25519

# Clone using SSH
git clone gitea:yourorg/yourrepo.git

# Or with full URL
git clone ssh://git@git.example.com:2222/yourorg/yourrepo.git
```

### Step 4: Team and Organization Setup

```bash
# Gitea CLI (tea) for scripted setup
brew install tea

# Login
tea login add \
  --name mycompany \
  --url https://git.example.com \
  --token your-api-token

# Create organization
tea org create mycompany

# Create team within org
tea org team create --org mycompany \
  --name "Developers" \
  --permission write \
  --units repo,issue,pullrequest

# Add members to team
tea org team user add --org mycompany --team Developers alice
tea org team user add --org mycompany --team Developers bob
```

### Step 5: Repository Templates

Create a template repo then:

```bash
# Via API: create repo from template
curl -X POST "https://git.example.com/api/v1/repos/mycompany/service-template/generate" \
  -H "Authorization: token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "owner": "mycompany",
    "name": "new-service",
    "description": "New microservice",
    "private": true,
    "git_content": true
  }'
```

### Step 6: Gitea Actions (CI/CD)

Gitea Actions uses the same syntax as GitHub Actions.

```bash
# Enable Actions in Gitea admin panel:
# Site Admin > Configuration > Actions > Enable

# Start an Actions runner
docker run -d \
  --name gitea-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ./runner:/data \
  -e GITEA_INSTANCE_URL=https://git.example.com \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=your-runner-token \
  --restart unless-stopped \
  gitea/act_runner:latest
```

```yaml
# .gitea/workflows/test.yml
name: Test and Lint

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Test
        run: go test ./...

      - name: Lint
        uses: golangci/golangci-lint-action@v3
```

### Step 7: Webhooks for Notifications

```bash
# Create webhook via API
curl -X POST "https://git.example.com/api/v1/repos/mycompany/myrepo/hooks" \
  -H "Authorization: token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "slack",
    "config": {
      "url": "https://hooks.slack.com/services/xxx",
      "channel": "#engineering",
      "username": "Gitea",
      "icon_url": "https://gitea.io/images/gitea.png"
    },
    "events": ["push", "pull_request", "issues"],
    "active": true
  }'
```

### Step 8: Backup Script

```bash
#!/bin/bash
# scripts/backup-gitea.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/gitea"

mkdir -p "$BACKUP_DIR"

# Dump Gitea config and repos via built-in tool
docker exec gitea gitea admin dump \
  --config /data/gitea/conf/app.ini \
  --file "/tmp/gitea-dump-${DATE}.zip" \
  --type zip

# Copy out of container
docker cp "gitea:/tmp/gitea-dump-${DATE}.zip" "$BACKUP_DIR/"

# Also dump database separately
docker exec gitea_db pg_dump \
  -U gitea gitea | gzip > "$BACKUP_DIR/gitea-db-${DATE}.sql.gz"

# Ship to object storage
mc cp "$BACKUP_DIR/gitea-dump-${DATE}.zip" company/backups/gitea/
mc cp "$BACKUP_DIR/gitea-db-${DATE}.sql.gz" company/backups/gitea/

# Remove local copies older than 7 days
find "$BACKUP_DIR" -mtime +7 -delete

echo "Gitea backup complete: gitea-dump-${DATE}.zip"
```

### Step 9: Branch Protection Rules

```bash
# Via API: protect main branch
curl -X POST "https://git.example.com/api/v1/repos/mycompany/myrepo/branch_protections" \
  -H "Authorization: token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "branch_name": "main",
    "enable_push": false,
    "enable_push_whitelist": true,
    "push_whitelist_teams": ["leads"],
    "require_signed_commits": false,
    "enable_status_check": true,
    "status_check_contexts": ["test", "lint"],
    "required_approvals": 1,
    "dismiss_stale_approvals": true
  }'
```

## Gitea API Automation

Gitea ships with a full REST API documented at `/swagger` on your instance. Teams use it for onboarding automation, repository templating, and dashboard integrations.

```bash
# List all repos in an org (paginated)
curl -s "https://git.example.com/api/v1/orgs/mycompany/repos?limit=50&page=1" \
  -H "Authorization: token your-api-token" | jq '.[].full_name'

# Mirror an external repo into Gitea (for archiving or vendoring)
curl -X POST "https://git.example.com/api/v1/repos/migrate" \
  -H "Authorization: token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "clone_addr": "https://github.com/upstream/project.git",
    "mirror": true,
    "mirror_interval": "8h0m0s",
    "repo_name": "project-mirror",
    "repo_owner": "mycompany",
    "private": true
  }'

# Create a deploy key on a repo (for CI runners)
curl -X POST "https://git.example.com/api/v1/repos/mycompany/myrepo/keys" \
  -H "Authorization: token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "ssh-ed25519 AAAA... ci-runner-key",
    "read_only": true,
    "title": "CI Runner"
  }'
```

### Scripted Onboarding

When a new developer joins, automate the full onboarding with a shell script:

```bash
#!/bin/bash
# scripts/onboard-dev.sh USERNAME EMAIL
set -e

USERNAME="$1"
EMAIL="$2"
API="https://git.example.com/api/v1"
TOKEN="$GITEA_ADMIN_TOKEN"

# Create user
curl -s -X POST "$API/admin/users" \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"email\": \"$EMAIL\",
    \"login_name\": \"$USERNAME\",
    \"username\": \"$USERNAME\",
    \"password\": \"ChangeMe123!\",
    \"must_change_password\": true,
    \"send_notify\": true
  }"

# Add to org teams
TEAM_ID=$(curl -s "$API/orgs/mycompany/teams" \
  -H "Authorization: token $TOKEN" | jq '.[] | select(.name=="Developers") | .id')

curl -s -X PUT "$API/teams/$TEAM_ID/members/$USERNAME" \
  -H "Authorization: token $TOKEN"

echo "Onboarded $USERNAME — password reset required on first login"
```

## Upgrading Gitea

Gitea follows semantic versioning. Minor upgrades (1.21.x → 1.21.y) are safe to do any time. Major upgrades require reading the release notes for migration steps.

```bash
# Pull the new image
docker compose pull server

# Back up before upgrading (always)
./scripts/backup-gitea.sh

# Apply the upgrade
docker compose up -d server

# Check logs for any migration output
docker compose logs -f server | grep -E "migration|error|panic"

# Verify version
curl -s https://git.example.com/api/v1/version | jq .version
```

Pin the image tag in your `docker-compose.yml` (e.g., `gitea/gitea:1.22.1`) rather than using `latest`. This prevents surprise upgrades when you run `docker compose pull` for unrelated reasons.

## Monitoring Gitea Health

```bash
# Gitea exposes metrics at /metrics (enable in app.ini)
# In docker-compose, add:
GITEA__metrics__ENABLED=true
GITEA__metrics__TOKEN=your-metrics-token

# Scrape from Prometheus
# prometheus.yml
scrape_configs:
  - job_name: gitea
    bearer_token: your-metrics-token
    static_configs:
      - targets: ['git.example.com:443']
    scheme: https
    metrics_path: /metrics
```

Useful Gitea metrics to alert on:

- `gitea_repositories_total` — track repo growth over time
- `gitea_users_total` — unexpected spikes may indicate account compromise
- `process_resident_memory_bytes` — Gitea is lean; spikes indicate runaway git operations
- `gitea_actions_runners` — ensure your CI runner count stays above zero

For a minimal health check endpoint, Gitea also provides `/api/v1/settings/api` which returns 200 when the instance is up and reachable. Add this to your uptime monitor (UptimeRobot, Uptime Kuma, or Grafana Synthetic Monitoring).

## Managing Multiple Runners and Labels

When your team grows, you will want runners with different capabilities — a runner with Docker-in-Docker for container builds, a runner with GPU access for ML tests, or a macOS runner for native builds. Gitea Actions supports runner labels to target specific machines:

```bash
# Register a second runner with a custom label
docker run -d \
  --name gitea-runner-macos \
  -e GITEA_INSTANCE_URL=https://git.example.com \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=your-runner-token \
  -e GITEA_RUNNER_LABELS=macos,native \
  --restart unless-stopped \
  gitea/act_runner:latest
```

Labels are matched at scheduling time. Workflows that require `ubuntu-latest` go to Linux runners; workflows requiring `macos` go to the macOS runner. If no matching runner is online, the job queues until one becomes available — Gitea does not fail the job immediately, giving runners time to reconnect after maintenance.

## Pull Request Review Workflow

Gitea's PR workflow is close to GitHub's but with a few differences in configuration worth knowing. Set sensible repository defaults via the API:

```bash
# Set squash-merge as default and auto-delete merged branches
curl -X PATCH "https://git.example.com/api/v1/repos/mycompany/myrepo" \
  -H "Authorization: token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    default_merge_style: squash,
    default_delete_branch_after_merge: true,
    default_allow_maintainer_edit: true
  }'
```

Configure `default_delete_branch_after_merge: true` to keep the branch list clean. With squash merging as default, your `main` history stays linear and readable, which matters when `git bisect` is your primary debugging tool during an incident. Combine this with the branch protection rule requiring at least one approval and passing status checks, and you have a review workflow that is safe without being bureaucratic.

## Related Reading

- [ADR Tools for Remote Engineering Teams](/remote-work-tools/adr-tools-for-remote-engineering-teams/)
- [How to Create Automated Deployment Notifications](/remote-work-tools/how-to-create-automated-deployment-notifications/)
- [Best Practice for Remote Team README Files in Repositories](/remote-work-tools/best-practice-for-remote-team-readme-files-in-repositories-s/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
