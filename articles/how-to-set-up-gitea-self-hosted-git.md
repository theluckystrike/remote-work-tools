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

## Docker Compose Deployment

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

## Nginx Reverse Proxy

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

## SSH Configuration for Team Members

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

## Team and Organization Setup

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

## Repository Templates

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

## Gitea Actions (CI/CD)

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

## Webhooks for Notifications

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

## Backup Script

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

## Branch Protection Rules

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

## Related Reading

- [ADR Tools for Remote Engineering Teams](/remote-work-tools/adr-tools-for-remote-engineering-teams/)
- [How to Create Automated Deployment Notifications](/remote-work-tools/how-to-create-automated-deployment-notifications/)
- [Best Practice for Remote Team README Files in Repositories](/remote-work-tools/best-practice-for-remote-team-readme-files-in-repositories-s/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
