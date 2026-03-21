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
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A portable dev environment solves the biggest friction in remote development: getting a new machine, a new teammate, or a new CI environment up and running in minutes instead of hours. Docker makes the environment a file that you check into version control alongside your code.

This guide builds a complete portable dev environment: a base Dockerfile, a Docker Compose setup with services, and a VS Code dev container config — all usable from any machine with Docker installed.

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



## Related Articles

- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)
- [Developer environment bootstrap script](/remote-work-tools/how-to-onboard-new-remote-employees-in-first-week-step-by-st/)
- [Remote Sales Team Demo Environment Setup for Distributed](/remote-work-tools/remote-sales-team-demo-environment-setup-for-distributed-sol/)
- [Asana vs Linear for a 10-Person Dev Team Comparison](/remote-work-tools/asana-vs-linear-for-a-10-person-dev-team-comparison/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
