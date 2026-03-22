---
layout: default
title: "How to Set Up Woodpecker CI for Self-Hosted"
description: "Deploy and configure Woodpecker CI for self-hosted continuous integration with Gitea, GitHub, or GitLab on your own infrastructure"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-woodpecker-ci-for-self-hosted/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Woodpecker CI is a lightweight, open-source CI system forked from Drone. It runs on your own hardware, integrates with Gitea, GitHub, GitLab, and Forgejo, and uses a simple YAML pipeline syntax. For remote teams that want GitHub Actions-style workflows without GitHub's pricing or data residency concerns, it's the cleanest self-hosted option.

---

## Architecture Overview

Woodpecker has two components:
- **Server**: Web UI, API, pipeline scheduler. Runs as a single container.
- **Agent**: Executes pipeline steps. Run one per host; scale horizontally.

Both communicate over gRPC. The server stores state in SQLite (small teams) or PostgreSQL (production).

---

## Deploy with Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  woodpecker-server:
    image: woodpeckerci/woodpecker-server:latest
    ports:
      - "8000:8000"
      - "9000:9000"
    volumes:
      - woodpecker-server-data:/var/lib/woodpecker/
    environment:
      - WOODPECKER_OPEN=false
      - WOODPECKER_HOST=https://ci.yourcompany.com
      - WOODPECKER_GITHUB=true
      - WOODPECKER_GITHUB_CLIENT=${GITHUB_CLIENT_ID}
      - WOODPECKER_GITHUB_SECRET=${GITHUB_CLIENT_SECRET}
      - WOODPECKER_AGENT_SECRET=${WOODPECKER_AGENT_SECRET}
      - WOODPECKER_DATABASE_DRIVER=postgres
      - WOODPECKER_DATABASE_DATASOURCE=postgres://woodpecker:${DB_PASSWORD}@postgres:5432/woodpecker?sslmode=disable
      - WOODPECKER_ADMIN=your-github-username
    depends_on:
      - postgres

  woodpecker-agent:
    image: woodpeckerci/woodpecker-agent:latest
    command: agent
    restart: always
    depends_on:
      - woodpecker-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WOODPECKER_SERVER=woodpecker-server:9000
      - WOODPECKER_AGENT_SECRET=${WOODPECKER_AGENT_SECRET}
      - WOODPECKER_MAX_WORKFLOWS=4
      - WOODPECKER_BACKEND=docker
      - WOODPECKER_BACKEND_DOCKER_NETWORK=ci-network

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=woodpecker
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=woodpecker
    volumes:
      - woodpecker-postgres-data:/var/lib/postgresql/data

volumes:
  woodpecker-server-data:
  woodpecker-postgres-data:
```

Create `.env`:

```bash
GITHUB_CLIENT_ID=your_oauth_app_client_id
GITHUB_CLIENT_SECRET=your_oauth_app_client_secret
WOODPECKER_AGENT_SECRET=$(openssl rand -hex 32)
DB_PASSWORD=$(openssl rand -hex 24)
```

---

## Create the GitHub OAuth App

1. Go to **GitHub > Settings > Developer settings > OAuth Apps > New OAuth App**
2. Set **Homepage URL**: `https://ci.yourcompany.com`
3. Set **Authorization callback URL**: `https://ci.yourcompany.com/authorize`
4. Copy the Client ID and generate a Client Secret

For Gitea instead of GitHub:

```bash
# docker-compose.yml environment
- WOODPECKER_GITHUB=false
- WOODPECKER_GITEA=true
- WOODPECKER_GITEA_URL=https://git.yourcompany.com
- WOODPECKER_GITEA_CLIENT=${GITEA_OAUTH_CLIENT_ID}
- WOODPECKER_GITEA_SECRET=${GITEA_OAUTH_CLIENT_SECRET}
```

---

## Nginx Reverse Proxy

```nginx
# /etc/nginx/sites-available/woodpecker
server {
    listen 443 ssl http2;
    server_name ci.yourcompany.com;

    ssl_certificate     /etc/letsencrypt/live/ci.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ci.yourcompany.com/privkey.pem;

    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        # Required for SSE (pipeline log streaming)
        proxy_buffering    off;
        proxy_cache        off;
        proxy_read_timeout 3600;
    }
}
```

---

## Write Your First Pipeline

Create `.woodpecker.yml` in your repo root:

```yaml
# .woodpecker.yml
steps:
  - name: test
    image: golang:1.22-alpine
    commands:
      - go test ./...

  - name: lint
    image: golangci/golangci-lint:latest
    commands:
      - golangci-lint run --timeout=5m

  - name: build
    image: golang:1.22-alpine
    commands:
      - CGO_ENABLED=0 go build -o ./bin/app ./cmd/app
    when:
      branch: main

  - name: docker-build
    image: plugins/docker
    settings:
      repo: yourcompany/app
      tags:
        - latest
        - ${CI_COMMIT_SHA:0:8}
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
    when:
      branch: main
      event: push
```

For Node.js:

```yaml
# .woodpecker.yml
steps:
  - name: install
    image: node:20-alpine
    commands:
      - npm ci

  - name: test
    image: node:20-alpine
    commands:
      - npm run test -- --coverage

  - name: build
    image: node:20-alpine
    commands:
      - npm run build
    when:
      branch: [main, staging]
```

---

## Secrets Management

Add secrets via the Woodpecker UI or CLI. Never put secrets in `.woodpecker.yml`:

```bash
# Install CLI
go install github.com/woodpecker-ci/woodpecker/cmd/woodpecker-cli@latest

# Authenticate
export WOODPECKER_SERVER=https://ci.yourcompany.com
export WOODPECKER_TOKEN=your_api_token

# Add a secret to a specific repo
woodpecker-cli secret add \
  --repository your-org/your-repo \
  --name docker_password \
  --value "your_registry_password"

# Add an organization-level secret (shared across repos)
woodpecker-cli secret add \
  --organization your-org \
  --name SLACK_WEBHOOK \
  --value "https://hooks.slack.com/..."
```

Reference in pipeline:

```yaml
steps:
  - name: deploy
    image: alpine
    environment:
      SLACK_WEBHOOK:
        from_secret: SLACK_WEBHOOK
    commands:
      - curl -X POST -d '{"text":"Deployed!"}' "$SLACK_WEBHOOK"
```

---

## Scale Agents Horizontally

Add agents on additional hosts by pointing them at the server:

```bash
docker run -d \
  --name woodpecker-agent-2 \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e WOODPECKER_SERVER=ci.yourcompany.com:9000 \
  -e WOODPECKER_AGENT_SECRET="$WOODPECKER_AGENT_SECRET" \
  -e WOODPECKER_MAX_WORKFLOWS=8 \
  woodpeckerci/woodpecker-agent:latest
```

For GPU workloads or specific hardware, tag agents:

```bash
# On agent startup
-e WOODPECKER_AGENT_LABELS="platform=linux,arch=arm64,gpu=true"
```

Target a specific agent in pipeline:

```yaml
steps:
  - name: ml-training
    image: pytorch/pytorch:latest
    labels:
      gpu: "true"
    commands:
      - python train.py
```

---

## Useful CLI Commands

```bash
# List pipelines for a repo
woodpecker-cli pipeline list --repository your-org/app

# Trigger a pipeline manually
woodpecker-cli pipeline start your-org/app --branch main

# View pipeline logs
woodpecker-cli pipeline log your-org/app 42

# List all repos
woodpecker-cli repo list

# Check agent status
woodpecker-cli agent list
```

---

## Matrix Builds

Test across multiple versions in parallel:

```yaml
# .woodpecker.yml
matrix:
  GO_VERSION:
    - "1.21"
    - "1.22"
    - "1.23"

steps:
  - name: test
    image: golang:${GO_VERSION}-alpine
    commands:
      - go test ./...
```

---

## Related Reading

- [How to Set Up Drone CI for Remote Teams](/remote-work-tools/how-to-set-up-drone-ci-for-remote-teams/)
- [Best Tools for Remote Team Code Ownership](/remote-work-tools/best-tools-remote-team-code-ownership/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)

---

## Related Articles

- [How to Set Up Drone CI for Remote Teams](/remote-work-tools/how-to-set-up-drone-ci-for-remote-teams/)
- [CI/CD Pipeline for Solo Developers: GitHub Actions](/remote-work-tools/ci-cd-pipeline-solo-developer-github-actions/)
- [Best Project Management Tools with GitHub Integration](/remote-work-tools/best-project-management-tools-with-github-integration/)
- [Best Tools for Remote Team Feature Flags](/remote-work-tools/best-tools-remote-team-feature-flags/)
- [Migrating from AWS CodeCommit to GitHub for Remote Team](/remote-work-tools/migrating-from-aws-codecommit-to-github-for-remote-team-code/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
