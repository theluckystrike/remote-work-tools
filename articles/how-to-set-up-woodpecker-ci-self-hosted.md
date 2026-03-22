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

The operational footprint is minimal: a server process and one or more agents. The server handles scheduling and the web interface. Agents execute pipeline steps in isolated Docker containers. Both communicate over gRPC. You can start with everything on one host and add agents on separate machines as build volume grows.

---

## Architecture Overview

Woodpecker has two components:
- **Server**: Web UI, API, pipeline scheduler. Runs as a single container.
- **Agent**: Executes pipeline steps. Run one per host; scale horizontally.

Both communicate over gRPC. The server stores state in SQLite (small teams) or PostgreSQL (production).

Each pipeline step runs in its own Docker container, pulled fresh for each build. Steps within a pipeline share a workspace volume so files written by one step (like a compiled binary) are available to the next. This model is simpler than GitHub Actions' runner model and easier to debug — you can reproduce any step locally by running the same Docker image with the same commands.

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

`WOODPECKER_OPEN=false` is important — it disables open registration so only users from your configured OAuth provider can log in. `WOODPECKER_ADMIN` grants admin access to the specified username, which lets you manage repositories and organization-level secrets from the UI.

Port 8000 is the HTTP/HTTPS interface. Port 9000 is the gRPC port that agents connect to. If you're behind a reverse proxy, only 8000 needs to be exposed externally — agents connect to 9000 over the internal Docker network.

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

For GitLab:

```bash
- WOODPECKER_GITHUB=false
- WOODPECKER_GITLAB=true
- WOODPECKER_GITLAB_URL=https://gitlab.yourcompany.com
- WOODPECKER_GITLAB_CLIENT=${GITLAB_APPLICATION_ID}
- WOODPECKER_GITLAB_SECRET=${GITLAB_APPLICATION_SECRET}
```

In GitLab, create the application at **User Settings > Applications** with the `api` and `read_user` scopes and the callback URL `https://ci.yourcompany.com/authorize`.

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

The `proxy_buffering off` and `proxy_read_timeout 3600` settings are not optional. Woodpecker streams pipeline logs to the browser using Server-Sent Events (SSE). Without buffering disabled, nginx buffers the stream and logs appear in chunks rather than in real time. Without a long timeout, nginx closes idle connections before long-running pipelines complete.

If you're using Traefik instead of nginx, add these labels to the server container:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.woodpecker.rule=Host(`ci.yourcompany.com`)"
  - "traefik.http.routers.woodpecker.entrypoints=websecure"
  - "traefik.http.routers.woodpecker.tls.certresolver=letsencrypt"
  - "traefik.http.services.woodpecker.loadbalancer.server.port=8000"
  - "traefik.http.middlewares.woodpecker-buf.buffering.maxResponseBodyBytes=0"
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

For Python with caching:

```yaml
steps:
  - name: install
    image: python:3.12-slim
    commands:
      - pip install --user -r requirements.txt

  - name: test
    image: python:3.12-slim
    commands:
      - python -m pytest tests/ -v --tb=short

  - name: type-check
    image: python:3.12-slim
    commands:
      - pip install --user mypy
      - python -m mypy src/
```

The `when` clause is the main conditional mechanism. You can filter on branch, event type (push, pull_request, tag), and step success/failure. A step with no `when` clause runs on every pipeline trigger.

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

Organization-level secrets are available to all repos within the organization without needing to add them per-repo. Useful for shared credentials like a container registry password or a deployment key that multiple services use. Repo-level secrets override organization-level secrets with the same name.

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

Agent labels are how you route specific pipeline steps to specific hardware. A common pattern: one agent on a fast x86 host for standard builds, one agent on an ARM host for cross-platform testing, one agent on a machine with an attached GPU for model training or inference tests. The scheduler matches steps to agents based on label intersection.

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

# Restart a failed pipeline
woodpecker-cli pipeline restart your-org/app 42

# Get pipeline info as JSON
woodpecker-cli pipeline info your-org/app 42 --output json
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

Woodpecker expands the matrix into parallel pipelines — one per version combination. If you add a second dimension:

```yaml
matrix:
  GO_VERSION:
    - "1.21"
    - "1.22"
  OS:
    - alpine
    - debian

steps:
  - name: test
    image: golang:${GO_VERSION}-${OS}
    commands:
      - go test ./...
```

This produces 4 pipelines (2 Go versions × 2 OS variants) running concurrently across available agents. Matrix builds are the cleanest way to validate compatibility without writing separate pipeline files for each combination.

---

## Conditional Pipelines with When Clauses

Control when entire pipelines or individual steps run:

```yaml
# Run only on pull requests targeting main
when:
  event: pull_request
  branch: main

# Run only when specific files change
when:
  path:
    include: ["src/**", "Dockerfile"]
    exclude: ["**/*.md"]

# Run on tag push (release builds)
when:
  event: tag
  tag: "v*"
```

Path filtering prevents unnecessary CI runs. A docs change that only touches Markdown files does not need to trigger a full build and test cycle. Combining path filtering with matrix builds keeps CI costs and wait times proportional to the scope of the change.

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
