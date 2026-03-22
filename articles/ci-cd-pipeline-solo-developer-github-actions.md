---
layout: default
title: "CI/CD Pipeline for Solo Developers: GitHub Actions"
description: "Set up a complete CI/CD pipeline as a solo developer using GitHub Actions. Covers linting, testing, Docker builds, and automated deploys to a VPS or cloud."
date: 2026-03-21
author: theluckystrike
permalink: /ci-cd-pipeline-solo-developer-github-actions/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A solo developer CI/CD pipeline does one thing: make sure you never manually deploy again. Every push to `main` runs your tests, builds your artifact, and ships it. If anything breaks, the deploy stops.

This guide builds a full pipeline using GitHub Actions: test on every PR, build a Docker image on merge to `main`, push to a registry, and deploy to a VPS via SSH. The same pattern works for static sites, Node apps, Python services, or Go binaries.

## What the Pipeline Does

```
push to PR branch
  → lint + test (fails fast)
  → PR passes checks

merge to main
  → lint + test
  → Docker build + push to GHCR
  → SSH into VPS, pull image, restart container
```

No manual steps. No "did I run tests?" anxiety.

## Repository Structure

```bash
myapp/
├── .github/
│   └── workflows/
│       ├── ci.yml          # runs on every push + PR
│       └── deploy.yml      # runs on merge to main
├── Dockerfile
├── docker-compose.prod.yml
├── src/
└── tests/
```

## CI Workflow: Test Every Push

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: ["**"]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
          retention-days: 7
```

This workflow runs on every push and every PR. If tests fail on a PR, the merge button is blocked. Branch protection rules enforce this — enable them under Settings → Branches → main → Require status checks.

## Deploy Workflow: Ship on Merge to Main

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            echo ${{ secrets.GHCR_TOKEN }} | docker login ghcr.io -u ${{ secrets.GHCR_USER }} --password-stdin
            docker pull ghcr.io/${{ github.repository }}:latest
            docker compose -f /opt/myapp/docker-compose.prod.yml up -d --no-deps app
            docker image prune -f
```

## Dockerfile for the App

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/
EXPOSE 3000
USER node
CMD ["node", "src/index.js"]
```

Multi-stage build keeps the image lean. The `builder` stage installs deps; the `runtime` stage copies only what runs in production.

## Production Docker Compose on the VPS

```yaml
# /opt/myapp/docker-compose.prod.yml
version: "3.9"

services:
  app:
    image: ghcr.io/youruser/myapp:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - NODE_ENV=production
    env_file:
      - .env.prod
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - app
```

The app binds only to `127.0.0.1:3000` — nginx is the public-facing proxy. This prevents direct container access from the internet.

## GitHub Secrets to Configure

Set these in Settings → Secrets and variables → Actions:

| Secret | Value |
|---|---|
| `VPS_HOST` | IP or domain of your VPS |
| `VPS_USER` | SSH user (e.g. `deploy`) |
| `VPS_SSH_KEY` | Private key (the full PEM, including headers) |
| `GHCR_USER` | Your GitHub username |
| `GHCR_TOKEN` | A PAT with `read:packages` scope |

`GITHUB_TOKEN` is automatic — no configuration needed.

## Setting Up the Deploy User on the VPS

```bash
# On your VPS — create a deploy user with minimal permissions
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy

# Add the GitHub Actions public key
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh

# Paste the public key (matching VPS_SSH_KEY secret)
sudo nano /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

# Give deploy user access to the app directory
sudo mkdir -p /opt/myapp
sudo chown deploy:deploy /opt/myapp
```

The deploy user has Docker access but no sudo. It can only pull images and restart containers.

## Adding a Database Migration Step

If your app uses database migrations, run them before the container swap:

```yaml
# Add this step before "Deploy to VPS" in deploy.yml
- name: Run migrations
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.VPS_HOST }}
    username: ${{ secrets.VPS_USER }}
    key: ${{ secrets.VPS_SSH_KEY }}
    script: |
      docker run --rm \
        --network host \
        --env-file /opt/myapp/.env.prod \
        ghcr.io/${{ github.repository }}:latest \
        node src/migrate.js
```

Migrations run in an one-off container using the same image before the service restarts.

## Caching Dependencies to Speed Up Builds

Docker layer caching via `cache-from: type=gha` reuses layers between runs. For npm specifically, the `actions/setup-node` cache key hashes `package-lock.json` — unchanged lock file means instant dep install.

For Python projects, replace the Node setup step:

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"
    cache-dependency-path: requirements.txt
```

## Notifications on Failure

```yaml
# Add to the end of deploy.yml
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "Deploy failed for ${{ github.repository }} on commit ${{ github.sha }}. <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View run>"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

This sends a Slack DM or channel message only when a deploy fails. No noise on success.

## Branch Protection Rules

Enable in Settings → Branches → Add rule for `main`:

- Require status checks: select the `test` job from `ci.yml`
- Require branches to be up to date before merging
- Require at least 1 approval (even solo: approve your own PRs with a second account, or disable for solo work)

With these rules, `main` is always green. A broken test cannot reach production.

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does GitHub offer a free tier?**

Most major tools offer some form of free tier or trial period. Check GitHub's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Example: GitHub Actions workflow for assessment tracking](/remote-work-tools/how-to-set-up-remote-hiring-pipeline-with-async-interviews-f/)
- [Example GitHub Actions quality gates](/remote-work-tools/how-to-coordinate-remote-frontend-developers-on-shared-compo/)
- [GitHub Actions Workflow for Remote Dev Teams](/remote-work-tools/github-actions-remote-dev-workflow/)
- [Best Free Tools for Solo Developer Managing Side Projects](/remote-work-tools/best-free-tools-for-solo-developer-managing-side-projects-re/)
- [Best Invoicing Workflow for Solo Developer with](/remote-work-tools/best-invoicing-workflow-for-solo-developer-with-international-clients/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
