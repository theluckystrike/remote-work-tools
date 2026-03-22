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

## Key Takeaways

- **It supports scoped packages**: htpasswd auth, S3 storage, and all package managers (npm, yarn, pnpm, bun).
- **Topics covered**: docker deployment, verdaccio configuration, user management
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## Docker Deployment

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

## Verdaccio Configuration

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

## User Management

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

## Nginx Reverse Proxy

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

## Developer Configuration

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

## Publishing Internal Packages

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

## CI/CD Publishing Workflow

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

## S3 Storage Backend

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

## Backup and Restore

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

## Related Reading

- [How to Set Up Gitea for Self-Hosted Git](/remote-work-tools/how-to-set-up-gitea-self-hosted-git/)
- [How to Set Up Renovate for Dependency Updates](/remote-work-tools/how-to-set-up-renovate-dependency-updates/)
- [Best Secrets Management Tool for Remote Dev Teams](/remote-work-tools/best-secrets-management-tool-for-remote-development-teams-us/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
