---
layout: default
title: "Portable Dev Environment with Docker 2026"
description: "Build a portable development environment with Docker that works identically on any machine. Covers Dockerfile, Compose, volume mounts, and dev container setup."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /portable-dev-environment-docker-2026/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools]---
---
layout: default
title: "Portable Dev Environment with Docker 2026"
description: "Build a portable development environment with Docker that works identically on any machine. Covers Dockerfile, Compose, volume mounts, and dev container setup."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /portable-dev-environment-docker-2026/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools]---

{% raw %}

A portable dev environment solves the biggest friction in remote development: getting a new machine, a new teammate, or a new CI environment up and running in minutes instead of hours. Docker makes the environment a file that you check into version control alongside your code.

This guide builds a complete portable dev environment: a base Dockerfile, a Docker Compose setup with services, and a VS Code dev container config — all usable from any machine with Docker installed.

## Key Takeaways

- **Free tiers typically have**: usage limits that work for evaluation but may not be sufficient for daily professional use.
- **Does Docker offer a**: free tier? Most major tools offer some form of free tier or trial period.
- **How do I get**: started quickly? Pick one tool from the options discussed and sign up for a free trial.
- **Focus on the 20%**: of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.
- **Use Alpine only if**: your project has no native dependencies.
- **Debian slim images are**: the practical default for most projects: small enough to pull quickly, glibc-compatible for native modules, and based on a well-supported OS with regular security patches.

## The Goal: One Command Setup

```bash
# Clone repo and start environment
git clone https://github.com/yourteam/myproject
cd myproject
docker compose up -d
# That's it — full dev environment running
```

## Writing a Good Dev Dockerfile

The base image choice matters. Use official language images from Docker Hub with a pinned version:

```dockerfile
# .devcontainer/Dockerfile
# Pin to specific version for reproducibility
FROM node:20.11.1-bookworm-slim

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    wget \
    ca-certificates \
    gnupg \
    sudo \
    vim \
    less \
    procps \
    htop \
    && rm -rf /var/lib/apt/lists/*

# Install additional dev tools
RUN npm install -g \
    typescript \
    ts-node \
    nodemon \
    @biomejs/biome

# Create non-root user (security best practice)
ARG USERNAME=developer
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# Set working directory
WORKDIR /workspace

# Switch to non-root user
USER $USERNAME

# Set up shell
RUN echo 'export PATH=$PATH:/workspace/node_modules/.bin' >> ~/.bashrc

# Keep container alive
CMD ["sleep", "infinity"]
```

For a Python project:

```dockerfile
# Dockerfile for Python dev environment
FROM python:3.12.2-slim-bookworm

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    make \
    build-essential \
    postgresql-client \
    redis-tools \
    && rm -rf /var/lib/apt/lists/*

# Install uv for fast package management
RUN pip install uv

ARG USERNAME=developer
RUN useradd -m -s /bin/bash $USERNAME \
    && echo "$USERNAME ALL=(root) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME

WORKDIR /workspace
USER $USERNAME

CMD ["sleep", "infinity"]
```

## Docker Compose for Full Stack Dev

A Compose file brings up your app and all its dependencies together:

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: .devcontainer/Dockerfile
    volumes:
      # Mount source code — changes reflect immediately
      - .:/workspace:cached
      # Persist node_modules inside container (faster than host mount)
      - node_modules:/workspace/node_modules
      # Mount SSH keys for git operations
      - ~/.ssh:/home/developer/.ssh:ro
      # Mount git config
      - ~/.gitconfig:/home/developer/.gitconfig:ro
    ports:
      - "3000:3000"
      - "9229:9229"  # Node debugger
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    stdin_open: true
    tty: true

  db:
    image: postgres:16.2-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"  # Expose for GUI tools
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7.2-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI

volumes:
  node_modules:
  postgres_data:
  redis_data:
```

## Dev Environment Scripts

Add a `Makefile` or shell scripts so the environment is one command to manage:

```makefile
# Makefile

.PHONY: up down build shell logs reset migrate test

# Start the environment
up:
	docker compose up -d
	@echo "Dev environment running. Connect with: make shell"

# Stop everything
down:
	docker compose down

# Rebuild images (after Dockerfile changes)
build:
	docker compose build --no-cache

# Open a shell in the app container
shell:
	docker compose exec app bash

# Follow app logs
logs:
	docker compose logs -f app

# Run database migrations
migrate:
	docker compose exec app npm run db:migrate

# Run tests
test:
	docker compose exec app npm test

# Full reset — destroy volumes and rebuild
reset:
	docker compose down -v
	docker compose build --no-cache
	docker compose up -d
	docker compose exec app npm run db:migrate
	docker compose exec app npm run db:seed

# Install dependencies
install:
	docker compose exec app npm install

# Production-like build
build-prod:
	docker build -f Dockerfile.prod -t myapp:latest .
```

## Persisting Data and Dotfiles

Named volumes in Compose persist database data between restarts. For dotfiles and editor configs inside the container:

```bash
# Bootstrap script to set up developer dotfiles inside the container
# .devcontainer/bootstrap.sh
#!/bin/bash
set -e

# Install personal dotfiles if available
if [ -d /home/developer/.dotfiles ]; then
  cd /home/developer/.dotfiles && ./install.sh
fi

# Install project dependencies
if [ -f /workspace/package.json ]; then
  cd /workspace && npm install
elif [ -f /workspace/requirements.txt ]; then
  cd /workspace && pip install -r requirements.txt
fi

echo "Dev environment bootstrapped."
```

Reference the bootstrap script in `devcontainer.json`:

```json
// .devcontainer/devcontainer.json
{
  "name": "Node.js 20 Dev",
  "dockerComposeFile": ["../docker-compose.yml"],
  "service": "app",
  "workspaceFolder": "/workspace",
  "postCreateCommand": "bash .devcontainer/bootstrap.sh",
  "customizations": {
    "vscode": {
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash",
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "biomejs.biome"
      },
      "extensions": [
        "biomejs.biome",
        "eamodio.gitlens",
        "ms-azuretools.vscode-docker",
        "ms-vscode.vscode-typescript-next"
      ]
    }
  },
  "remoteUser": "developer",
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {}
  }
}
```

## Managing Multiple Projects

When you have multiple projects, each with their own environment, port conflicts become an issue. Use a port convention:

```bash
# ~/.zshrc or ~/.bashrc

# Function to start a project's dev environment
dev() {
  local project=${1:-$(basename $PWD)}
  cd ~/projects/$project 2>/dev/null || true
  docker compose up -d
  echo "Started $project dev environment"
  docker compose ps
}

# Function to list all running dev environments
devls() {
  docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep -v "^NAMES"
}

# Stop all dev environments
devstop() {
  docker ps -q | xargs -r docker stop
  echo "All containers stopped"
}
```

Port convention example for preventing conflicts across projects:

```yaml
# Project A: ports in 3000-3099 range
# Project B: ports in 3100-3199 range
# Project C: ports in 3200-3299 range
```

## Multi-Architecture Builds (Apple Silicon + Linux CI)

Apple Silicon Macs (M1/M2/M3) and x86_64 Linux CI runners can hit compatibility problems if images are built for only one architecture. Use Docker Buildx to produce multi-platform images:

```bash
# Create a builder that supports multi-arch
docker buildx create --name multiarch --driver docker-container --use

# Build and push for both architectures simultaneously
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag yourregistry/myapp-dev:latest \
  --push \
  .devcontainer/

# Verify both architectures are present
docker buildx imagetools inspect yourregistry/myapp-dev:latest
```

When publishing to GitHub Container Registry (ghcr.io), authenticate with a personal access token that has `write:packages` scope:

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag ghcr.io/yourorg/myapp-dev:latest \
  --push \
  .devcontainer/
```

Team members on either architecture pull the same image tag and get the correct binary. This eliminates the "works on my Mac, fails in CI" class of problems.

## CI/CD Integration

The dev container configuration doubles as a CI specification. GitHub Actions can run tests inside the same container used for local development:

```yaml
# .github/workflows/test.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/yourorg/myapp-dev:latest
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    services:
      postgres:
        image: postgres:16.2-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: myapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@postgres:5432/myapp_test
        run: npm test
```

The CI environment is now identical to the local dev environment. A test that passes locally will pass in CI because both run the same container image.

## Choosing a Base Image: Comparison

Base image choice has downstream effects on image size, security surface, and available tools:

| Base Image | OS | Size (approx.) | glibc | Best For |
|---|---|---|---|---|
| `node:20-bookworm-slim` | Debian 12 slim | ~180MB | Yes | Node apps needing native modules |
| `node:20-alpine` | Alpine Linux | ~50MB | No (musl) | Minimal images, pure JS projects |
| `python:3.12-slim-bookworm` | Debian 12 slim | ~130MB | Yes | Python with C extensions |
| `python:3.12-alpine` | Alpine Linux | ~45MB | No (musl) | Lightweight pure Python |
| `ubuntu:24.04` | Ubuntu LTS | ~80MB | Yes | General-purpose, familiar toolchain |
| `mcr.microsoft.com/devcontainers/base:ubuntu` | Ubuntu LTS | ~420MB | Yes | VS Code dev containers with pre-installed tools |

**Alpine images** are smallest but use musl libc instead of glibc. Some native Node modules (sharp, bcrypt) and Python C extensions require glibc and will fail to compile on Alpine. Use Alpine only if your project has no native dependencies.

**Debian slim images** are the practical default for most projects: small enough to pull quickly, glibc-compatible for native modules, and based on a well-supported OS with regular security patches.

**Microsoft's devcontainers base images** (`mcr.microsoft.com/devcontainers/`) come pre-installed with git, zsh, and VS Code server integration. They are larger but save the effort of scripting developer tooling from scratch.

## Keeping Images Up to Date

Pinned image versions (e.g., `node:20.11.1-bookworm-slim`) prevent surprise breakage but require deliberate updates. Automate this with Dependabot:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: docker
    directory: "/"
    schedule:
      interval: weekly
    labels:
      - dependencies
      - docker

  - package-ecosystem: docker
    directory: "/.devcontainer"
    schedule:
      interval: weekly
```

Dependabot opens a PR each week when newer patch versions of your pinned images are available. Review the changelog, run CI, and merge if green — no manual version tracking required.

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does Docker offer a free tier?**

Most major tools offer some form of free tier or trial period. Check Docker's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)
- [Developer environment bootstrap script](/remote-work-tools/how-to-onboard-new-remote-employees-in-first-week-step-by-st/)
- [Remote Sales Team Demo Environment Setup for Distributed](/remote-work-tools/remote-sales-team-demo-environment-setup-for-distributed-sol/)
- [Asana vs Linear for a 10-Person Dev Team Comparison](/remote-work-tools/asana-vs-linear-for-a-10-person-dev-team-comparison/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
