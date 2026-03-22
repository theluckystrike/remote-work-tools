---
layout: default
title: "How to Set Up Verdaccio Private npm Registry"
description: "Deploy Verdaccio as a private npm registry for remote teams to publish internal packages with scoped access, S3 storage, and npm/yarn/pnpm support"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-verdaccio-private-npm-registry/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Verdaccio is a lightweight Node.js private npm registry that proxies the public npm registry and lets your team publish internal packages. It supports scoped packages, htpasswd auth, S3 storage, and all package managers (npm, yarn, pnpm, bun). This guide deploys it with Docker and configures team publishing workflows.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Verdaccio Plugins for Team Workflows](#verdaccio-plugins-for-team-workflows)
- [Scoped Package Strategy for Large Teams](#scoped-package-strategy-for-large-teams)
- [Monitoring Verdaccio](#monitoring-verdaccio)
- [Caching Strategy and Offline Resilience](#caching-strategy-and-offline-resilience)
- [Verdaccio vs. Alternatives](#verdaccio-vs-alternatives)
- [Related Reading](#related-reading)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Docker Deployment

```yaml
# docker-compose.yml
version: "3.8"

services:
  verdaccio:
    image: verdaccio/verdaccio:5
    container_name: verdaccio
    environment:
      - VERDACCIO_PUBLIC_URL=https://npm.example.com
    volumes:
      - ./verdaccio/config:/verdaccio/conf
      - ./verdaccio/storage:/verdaccio/storage
      - ./verdaccio/plugins:/verdaccio/plugins
    ports:
      - "4873:4873"
    restart: unless-stopped
```

```bash
# Create config directory
mkdir -p verdaccio/config verdaccio/storage verdaccio/plugins
sudo chown -R 10001:65533 verdaccio/
```

### Step 2: Verdaccio Configuration

```yaml
# verdaccio/config/config.yaml
storage: /verdaccio/storage
auth:
  htpasswd:
    file: /verdaccio/conf/htpasswd
    max_users: 100
    algorithm: bcrypt
    rounds: 10

uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    cache: true
    timeout: 30s
    max_fails: 3
    fail_timeout: 5m

packages:
  # Private scoped packages - only your team can access/publish
  "@acme/*":
    access: authenticated
    publish: authenticated
    unpublish: authenticated

  # Read-only public mirror - authenticated users can read, nobody publishes
  "@types/*":
    access: authenticated
    proxy: npmjs

  "**":
    access: authenticated
    proxy: npmjs
    unpublish: authenticated

server:
  keepAliveTimeout: 60

middlewares:
  audit:
    enabled: true

logs:
  - { type: stdout, format: pretty, level: http }

security:
  api:
    legacy: true
    jwt:
      sign:
        expiresIn: 30d
      verify:
        someProp: [secret]
  web:
    sign:
      expiresIn: 7d

web:
  title: "ACME npm Registry"
  enable: true
  primary_color: "#4D4D4D"
  scope: "@acme"
```

### Step 3: User Management

```bash
# Install verdaccio CLI
npm install -g verdaccio

# Add users via htpasswd
docker exec verdaccio htpasswd -B -b /verdaccio/conf/htpasswd alice alicepassword
docker exec verdaccio htpasswd -B -b /verdaccio/conf/htpasswd bob bobpassword
docker exec verdaccio htpasswd -B -b /verdaccio/conf/htpasswd ci-runner cipassword

# Self-service via npm (if registration is enabled)
npm adduser --registry https://npm.example.com
```

### Step 4: Nginx Reverse Proxy

```nginx
# /etc/nginx/sites-available/verdaccio
server {
    listen 80;
    server_name npm.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name npm.example.com;

    ssl_certificate /etc/letsencrypt/live/npm.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/npm.example.com/privkey.pem;

    client_max_body_size 100m;

    location / {
        proxy_pass http://localhost:4873;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-NginX-Proxy true;
    }
}
```

### Step 5: Developer Configuration

Each developer configures their npm to use the private registry:

```bash
# Method 1: .npmrc in project root (recommended, committed to git)
# .npmrc
registry=https://npm.example.com/
@acme:registry=https://npm.example.com/
//npm.example.com/:_authToken=${NPM_TOKEN}
always-auth=false

# Method 2: Global npm config
npm config set registry https://npm.example.com
npm config set @acme:registry https://npm.example.com

# Login
npm login --registry https://npm.example.com
# Username: alice
# Password: alicepassword
# Email: alice@example.com

# Verify
npm whoami --registry https://npm.example.com
```

```bash
# pnpm configuration
# .npmrc (pnpm reads the same file)
@acme:registry=https://npm.example.com/
//npm.example.com/:_authToken=${NPM_TOKEN}

# yarn .yarnrc.yml
npmRegistries:
  "https://npm.example.com":
    npmAuthToken: "${NPM_TOKEN}"

npmScopes:
  acme:
    npmRegistryServer: "https://npm.example.com"
```

### Step 6: Publish Internal Packages

```json
// packages/ui-components/package.json
{
  "name": "@acme/ui-components",
  "version": "1.0.0",
  "description": "Shared UI component library",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "publishConfig": {
    "registry": "https://npm.example.com",
    "access": "restricted"
  },
  "scripts": {
    "build": "tsc && vite build",
    "prepublishOnly": "npm run build"
  }
}
```

```bash
# Build and publish
cd packages/ui-components
npm run build
npm publish

# Verify it's available
npm info @acme/ui-components --registry https://npm.example.com

# Install in another project
npm install @acme/ui-components
```

### Step 7: Configure CI/CD Publishing Workflow

```yaml
# .github/workflows/publish.yml
name: Publish Package

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.example.com'

      - name: Install dependencies
        run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: Build
        run: npm run build

      - name: Publish
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Step 8: S3 Storage Backend

For production with multiple replicas, use S3 instead of local filesystem:

```bash
# Install S3 storage plugin
docker exec verdaccio npm install -g verdaccio-aws-s3-storage
```

```yaml
# config.yaml: replace storage section
store:
  aws-s3-storage:
    bucket: your-npm-registry-bucket
    region: us-east-1
    keyPrefix: verdaccio/
    endpoint: https://storage.example.com  # or remove for AWS S3
    s3ForcePathStyle: true  # Required for MinIO

# Set env vars:
# AWS_ACCESS_KEY_ID=your-access-key
# AWS_SECRET_ACCESS_KEY=your-secret
```

### Step 9: Backup and Restore

```bash
#!/bin/bash
# scripts/backup-verdaccio.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="/backups/verdaccio-${DATE}.tar.gz"

tar czf "$BACKUP_PATH" ./verdaccio/storage ./verdaccio/config

# Ship to MinIO
mc cp "$BACKUP_PATH" company/backups/verdaccio/

echo "Verdaccio backup: $BACKUP_PATH"
```

```bash
# Restore
tar xzf "/backups/verdaccio-20260322_020000.tar.gz"
docker compose restart verdaccio
```

## Verdaccio Plugins for Team Workflows

Verdaccio's plugin system extends its capabilities well beyond basic auth and storage. The most useful plugins for remote teams are:

**verdaccio-github-oauth-ui** — Replaces htpasswd with GitHub OAuth login, so developers authenticate with their GitHub accounts and token rotation is automatic. Configuration is minimal: set the GitHub OAuth app credentials and the registry handles the rest.

**verdaccio-audit** — Enables `npm audit` against your private registry by proxying the npm audit endpoint. Developers running `npm audit` in a project that uses private packages get results from both the public advisory database and your registry's metadata.

**verdaccio-ldap** — Connects to an existing corporate LDAP or Active Directory for authentication, which avoids managing a separate htpasswd user database when you already have an identity provider.

Installing a plugin requires placing it in the plugins volume directory and referencing it in config:

```bash
# Add plugin to running container (for testing)
docker exec verdaccio npm install verdaccio-github-oauth-ui

# Or add to Dockerfile for a custom image
FROM verdaccio/verdaccio:5
RUN npm install -g verdaccio-github-oauth-ui
```

```yaml
# config.yaml — GitHub OAuth auth section
auth:
  github-oauth-ui:
    client-id: your-github-app-client-id
    client-secret: your-github-app-client-secret
    org: your-github-org
```

## Scoped Package Strategy for Large Teams

Flat package names in a private registry become hard to manage as teams grow. A scoped namespace strategy keeps packages discoverable and enforces ownership:

```
@acme/ui-*        — Frontend design system and shared components (owned by UI team)
@acme/api-*       — Shared API clients and SDK wrappers (owned by Platform team)
@acme/config-*    — Shared ESLint, TypeScript, and build configs
@acme/shared-*    — Cross-team utilities (any team can publish, PR required)
```

Enforce this in Verdaccio config by giving each scope a separate access rule:

```yaml
packages:
  "@acme/ui-*":
    access: authenticated
    publish: ui-team
    unpublish: ui-team

  "@acme/api-*":
    access: authenticated
    publish: platform-team
    unpublish: platform-team

  "@acme/config-*":
    access: authenticated
    publish: platform-team
    unpublish: platform-team

  "@acme/*":
    access: authenticated
    publish: authenticated
    unpublish: authenticated
```

This requires using `verdaccio-htpasswd-groups` or a plugin that understands user groups. With plain htpasswd, all authenticated users can publish to any scope — the pattern above enforces per-scope ownership only with group-aware auth.

## Monitoring Verdaccio

Verdaccio exposes basic metrics at `/-/ping` and logs HTTP traffic to stdout. For production, ship logs to your observability stack and set up an uptime check:

```yaml
# docker-compose.yml — add healthcheck
services:
  verdaccio:
    image: verdaccio/verdaccio:5
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:4873/-/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
```

For richer metrics, pair Verdaccio with a Loki log aggregation setup: Verdaccio's `http`-level logs capture every install, publish, and auth event with timestamps. A simple Grafana dashboard tracking publish frequency, install rates, and auth failures gives your platform team visibility into registry health without custom instrumentation.

## Caching Strategy and Offline Resilience

One of the most underused Verdaccio features for remote teams is its aggressive caching of public registry packages. When a developer installs a package routed through Verdaccio, the tarball is stored locally under `verdaccio/storage`. Subsequent installs of the same version — from any developer's machine or CI runner — hit the local cache without reaching npmjs.org.

This matters for three reasons. First, it eliminates dependency on npm's CDN uptime; a registry outage does not break your builds. Second, it makes CI pipelines faster: a cold runner installing `react` from a Verdaccio cache on your LAN is faster than pulling from a remote CDN. Third, it freezes public package versions at the point they were first installed, so you cannot silently get a different tarball for the same version string later.

Configure the uplink timeout aggressively to prefer cache over network:

```yaml
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    cache: true
    timeout: 10s
    max_fails: 2
    fail_timeout: 10m
    maxage: 30m    # Cache metadata for 30 minutes before re-fetching
```

To pre-warm the cache for critical packages before a deploy, install them through Verdaccio from a script:

```bash
#!/bin/bash
# scripts/warm-verdaccio-cache.sh
REGISTRY=https://npm.example.com
PACKAGES=(
  "react@18.2.0"
  "react-dom@18.2.0"
  "@types/react@18.2.0"
  "typescript@5.3.3"
  "vite@5.1.0"
)

for pkg in "${PACKAGES[@]}"; do
  npm pack "${pkg}" --registry "${REGISTRY}" --dry-run
  echo "Cached: ${pkg}"
done
```

The `npm pack --dry-run` forces Verdaccio to fetch and cache the tarball without writing anything locally. After this runs, CI runners pulling those exact versions will get them from the local cache consistently.

## Verdaccio vs. Alternatives

Verdaccio is the right choice for teams that want a self-hosted registry with zero vendor dependency and minimal infrastructure cost. It runs on a single Docker container, uses local filesystem storage by default, and has no external service dependencies for basic operation.

The trade-off compared to managed alternatives:

- **vs. npm Organizations (npmjs.com)**: npm Orgs is simpler to set up and requires no infrastructure, but you pay per seat and all packages live on the public internet. Verdaccio keeps packages fully private with no external exposure.
- **vs. GitHub Packages (GHCR for npm)**: GitHub Packages is convenient if you are already on GitHub, but package visibility is tied to repo visibility, and download bandwidth costs can add up at scale. Verdaccio has no per-download cost.
- **vs. Artifactory/Nexus**: Both support npm registries with enterprise features (LDAP, HA, auditing), but they are significantly heavier and require paid licenses for production features. Verdaccio covers 90% of what most teams need without the operational burden.

For a team of 5-50 developers publishing a handful of internal packages, Verdaccio is the practical choice. When you need HA, cross-format support (Maven, PyPI, Docker in one tool), and enterprise RBAC, Nexus or Artifactory become worth the complexity.

## Related Reading

- [How to Set Up Gitea for Self-Hosted Git](/remote-work-tools/how-to-set-up-gitea-self-hosted-git/)
- [How to Set Up Renovate for Dependency Updates](/remote-work-tools/how-to-set-up-renovate-dependency-updates/)
- [Best Secrets Management Tool for Remote Dev Teams](/remote-work-tools/best-secrets-management-tool-for-remote-development-teams-us/)

---

## Related Articles

- [Node.js and npm](/remote-work-tools/claude-code-npm-package-development-guide/)
- [Setting Up Harbor for Container Registry](/remote-work-tools/setting-up-harbor-for-container-registry/)
- [Best Container Registry Tool for Remote Teams Sharing](/remote-work-tools/best-container-registry-tool-for-remote-teams-sharing-docker/)
- [Remote Team Charter Template Guide 2026](/remote-work-tools/remote-team-charter-template-guide-2026/)
- [How to Secure Slack and Teams Channels for Remote Team](/remote-work-tools/how-to-secure-slack-and-teams-channels-for-remote-team-confi/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
