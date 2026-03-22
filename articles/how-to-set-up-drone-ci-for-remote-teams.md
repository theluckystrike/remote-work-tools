---
layout: default
title: "How to Set Up Drone CI for Remote Teams"
description: "Deploy Drone CI on your own infrastructure for self-hosted continuous integration with Docker-native pipelines and multi-architecture builds"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-drone-ci-for-remote-teams/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Drone CI is a container-native CI system where every pipeline step runs in a Docker container. There's no plugin system to fight with, no shared state between steps by default, and pipeline configs are just YAML that any developer can understand. For remote teams that self-host, Drone's simplicity reduces the operational burden compared to Jenkins.

---

## Architecture

Drone has two components:
- **Server**: Manages the web UI, API, and pipeline queue. One instance.
- **Runner**: Executes pipeline steps on Docker. Run as many as needed.

Runners communicate with the server over HTTP. The server stores state in SQLite or PostgreSQL.

---

## Deploy with Docker Compose

For GitHub integration:

```yaml
# docker-compose.yml
version: "3.8"

services:
  drone-server:
    image: drone/drone:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/lib/drone:/data
    restart: always
    environment:
      - DRONE_GITHUB_CLIENT_ID=${GITHUB_CLIENT_ID}
      - DRONE_GITHUB_CLIENT_SECRET=${GITHUB_CLIENT_SECRET}
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_SERVER_HOST=drone.yourcompany.com
      - DRONE_SERVER_PROTO=https
      - DRONE_TLS_AUTOCERT=true
      - DRONE_USER_CREATE=username:your-github-username,admin:true
      - DRONE_DATABASE_DRIVER=postgres
      - DRONE_DATABASE_DATASOURCE=postgres://drone:${DB_PASSWORD}@postgres:5432/drone?sslmode=disable
    depends_on:
      - postgres

  drone-runner:
    image: drone/drone-runner-docker:1
    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_RPC_PROTO=https
      - DRONE_RPC_HOST=drone.yourcompany.com
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_RUNNER_CAPACITY=4
      - DRONE_RUNNER_NAME=runner-01
      - DRONE_RUNNER_LABELS=platform:linux,arch:amd64

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=drone
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=drone
    volumes:
      - drone-postgres-data:/var/lib/postgresql/data

volumes:
  drone-postgres-data:
```

Create `.env`:

```bash
GITHUB_CLIENT_ID=your_oauth_app_id
GITHUB_CLIENT_SECRET=your_oauth_app_secret
DRONE_RPC_SECRET=$(openssl rand -hex 32)
DB_PASSWORD=$(openssl rand -hex 24)
```

For Gitea instead of GitHub:

```bash
# Replace GitHub env vars with:
- DRONE_GITEA_SERVER=https://git.yourcompany.com
- DRONE_GITEA_CLIENT_ID=${GITEA_CLIENT_ID}
- DRONE_GITEA_CLIENT_SECRET=${GITEA_CLIENT_SECRET}
```

---

## Write Your First Pipeline

Create `.drone.yml` in your repository root:

```yaml
# .drone.yml
kind: pipeline
type: docker
name: default

steps:
  - name: test
    image: golang:1.22-alpine
    commands:
      - go test ./...
      - go vet ./...

  - name: build
    image: golang:1.22-alpine
    commands:
      - CGO_ENABLED=0 go build -o ./bin/app ./cmd/app
    when:
      branch: main

  - name: docker-build-push
    image: plugins/docker
    settings:
      repo: yourcompany/app
      auto_tag: true
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
    when:
      branch: main
      event: push

  - name: notify
    image: plugins/slack
    settings:
      webhook:
        from_secret: slack_webhook
      channel: "#deployments"
      template: >
        {{#success build.status}}
          Build {{build.number}} succeeded on {{build.branch}}
        {{else}}
          Build {{build.number}} failed on {{build.branch}}
        {{/success}}
    when:
      status: [success, failure]
```

For Node.js:

```yaml
kind: pipeline
type: docker
name: node-app

steps:
  - name: install
    image: node:20-alpine
    commands:
      - npm ci
      - npm run build

  - name: test
    image: node:20-alpine
    commands:
      - npm run test:ci
    environment:
      NODE_ENV: test

  - name: e2e
    image: node:20-alpine
    commands:
      - npm run test:e2e
    when:
      branch: [main, staging]
```

---

## Manage Secrets

Add secrets via the Drone CLI:

```bash
# Install CLI
curl -L https://github.com/harness/drone-cli/releases/latest/download/drone_linux_amd64.tar.gz | tar zx
install -t /usr/local/bin drone

# Authenticate
export DRONE_SERVER=https://drone.yourcompany.com
export DRONE_TOKEN=your_drone_token  # Get from your user settings in UI

# Add a repo-level secret
drone secret add \
  --repository your-org/your-repo \
  --name docker_password \
  --data "your_registry_password"

# Add an organization-level secret
drone orgsecret add your-org slack_webhook "https://hooks.slack.com/..."
```

Reference secrets in `.drone.yml`:

```yaml
steps:
  - name: deploy
    image: alpine
    environment:
      DEPLOY_KEY:
        from_secret: deploy_key
    commands:
      - echo "$DEPLOY_KEY" | ssh-add -
      - ssh deploy@prod.yourcompany.com "cd /app && git pull && pm2 restart app"
```

---

## Multi-Architecture Builds

Build for AMD64 and ARM64 in parallel using Drone's multi-pipeline support:

```yaml
# .drone.yml
---
kind: pipeline
type: docker
name: linux-amd64

platform:
  os: linux
  arch: amd64

steps:
  - name: build
    image: golang:1.22-alpine
    commands:
      - GOOS=linux GOARCH=amd64 go build -o ./bin/app-amd64 ./cmd/app

  - name: push
    image: plugins/docker
    settings:
      repo: yourcompany/app
      tags: linux-amd64-${DRONE_COMMIT_SHA:0:8}
      username: { from_secret: docker_username }
      password: { from_secret: docker_password }
    when:
      branch: main

---
kind: pipeline
type: docker
name: linux-arm64

platform:
  os: linux
  arch: arm64

steps:
  - name: build
    image: golang:1.22-alpine
    commands:
      - GOOS=linux GOARCH=arm64 go build -o ./bin/app-arm64 ./cmd/app

  - name: push
    image: plugins/docker
    settings:
      repo: yourcompany/app
      tags: linux-arm64-${DRONE_COMMIT_SHA:0:8}
      username: { from_secret: docker_username }
      password: { from_secret: docker_password }
    when:
      branch: main

---
kind: pipeline
type: docker
name: manifest

depends_on:
  - linux-amd64
  - linux-arm64

steps:
  - name: create-manifest
    image: plugins/manifest
    settings:
      username: { from_secret: docker_username }
      password: { from_secret: docker_password }
      target: yourcompany/app:latest
      template: yourcompany/app:linux-ARCH-${DRONE_COMMIT_SHA:0:8}
      platforms:
        - linux/amd64
        - linux/arm64
    when:
      branch: main
```

---

## Caching Dependencies

Cache node_modules or Go module cache between builds to speed up pipelines:

```yaml
steps:
  - name: restore-cache
    image: drillster/drone-volume-cache
    settings:
      restore: true
      mount:
        - ./node_modules
    volumes:
      - name: cache
        path: /cache

  - name: install
    image: node:20-alpine
    commands:
      - npm ci

  - name: rebuild-cache
    image: drillster/drone-volume-cache
    settings:
      rebuild: true
      mount:
        - ./node_modules
    volumes:
      - name: cache
        path: /cache

volumes:
  - name: cache
    host:
      path: /tmp/drone-cache
```

---

## Useful CLI Commands

```bash
# List builds for a repo
drone build ls your-org/your-repo

# View build details
drone build info your-org/your-repo 42

# Trigger a build on main
drone build create your-org/your-repo --branch main

# View logs for a specific step
drone log view your-org/your-repo 42 1 1

# List secrets for a repo
drone secret ls --repository your-org/your-repo
```

---

## Related Reading

- [How to Set Up Woodpecker CI for Self-Hosted](/remote-work-tools/how-to-set-up-woodpecker-ci-for-self-hosted/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [How to Create Automated Canary Deployments](/remote-work-tools/how-to-create-automated-canary-deployments/)

- [Async Decision-Making Framework for Remote Teams](/remote-work-tools/articles/how-to-set-up-async-decision-making-framework-guide/)
---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
