---
layout: default
title: "How to Create a Remote Dev Environment Template"
description: "Build a reproducible dev environment template with Devcontainers, Nix flakes, or a Makefile bootstrap that works for every remote team member"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-a-remote-dev-environment-template/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

"It works on my machine" becomes "it works on your machine too" when dev environments are codified. Remote teams are especially exposed to environment drift — developers are on different OS versions, different tool versions, and different configurations. A template that runs consistently on day one means less async debugging and faster onboarding.

---

## Option 1: Dev Containers (VS Code / JetBrains)

Dev Containers run your entire development environment inside a Docker container. VS Code and JetBrains connect to it ; the developer's local machine is just a display layer.

Create `.devcontainer/devcontainer.json` in your repo:

```json
{
  "name": "Project Dev Environment",
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "runArgs": [
    "--cap-add=SYS_PTRACE",
    "--security-opt", "seccomp=unconfined"
  ],
  "mounts": [
    "source=${localEnv:HOME}/.ssh,target=/home/vscode/.ssh,type=bind,readonly",
    "source=${localEnv:HOME}/.gitconfig,target=/home/vscode/.gitconfig,type=bind,readonly"
  ],
  "forwardPorts": [3000, 5432, 6379, 8080],
  "postCreateCommand": "make setup",
  "customizations": {
    "vscode": {
      "extensions": [
        "golang.go",
        "esbenp.prettier-vscode",
        "dbaeumer.vscode-eslint",
        "bradlc.vscode-tailwindcss",
        "ms-azuretools.vscode-docker"
      ],
      "settings": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "[go]": {
          "editor.defaultFormatter": "golang.go"
        }
      }
    }
  },
  "remoteUser": "vscode"
}
```

Create `.devcontainer/Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/devcontainers/base:ubuntu-22.04

# Install language runtimes
RUN apt-get update && apt-get install -y \
    curl wget git build-essential \
    postgresql-client redis-tools \
    && rm -rf /var/lib/apt/lists/*

# Go
ARG GO_VERSION=1.22.3
RUN curl -fsSL https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz \
    | tar -C /usr/local -xz
ENV PATH="/usr/local/go/bin:${PATH}"

# Node.js via nvm
ARG NODE_VERSION=20
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - \
    && apt-get install -y nodejs

# Tools
RUN go install github.com/air-verse/air@latest \
    && go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Set up non-root user
USER vscode
RUN curl -fsSL https://get.pnpm.io/install.sh | sh -
```

For teams using Docker Compose (multiple services):

```json
{
  "name": "Full Stack Dev",
  "dockerComposeFile": [
    "../docker-compose.yml",
    "docker-compose.devcontainer.yml"
  ],
  "service": "app",
  "workspaceFolder": "/workspace",
  "postCreateCommand": "make setup"
}
```

```yaml
# .devcontainer/docker-compose.devcontainer.yml
version: "3.8"
services:
  app:
    volumes:
      - ../..:/workspace:cached
    command: sleep infinity  # Keep container alive
```

---

## Option 2: Nix Flakes (Reproducible Across All OS)

Nix flakes provide bit-for-bit reproducible environments. The same `flake.nix` produces identical tool versions on macOS, Linux, and in CI.

```bash
# Install Nix (macOS/Linux)
curl --proto '=https' --tlsv1.2 -sSf https://install.determinate.systems/nix | sh -s -- install
```

Create `flake.nix` in your repo root:

```nix
{
  description = "Project development environment";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = import nixpkgs { inherit system; };
      in
      {
        devShells.default = pkgs.mkShell {
          buildInputs = with pkgs; [
            # Go toolchain
            go_1_22
            gopls
            golangci-lint
            air

            # Node.js
            nodejs_20
            nodePackages.pnpm

            # Database tools
            postgresql_16
            redis

            # Infrastructure
            terraform
            kubectl
            helm
            awscli2

            # Dev tools
            git
            gnumake
            jq
            yq
          ];

          shellHook = ''
            echo "Dev environment loaded"
            echo "Go: $(go version)"
            echo "Node: $(node --version)"

            # Set up local environment variables
            export GOPATH="$HOME/.local/share/go"
            export PATH="$GOPATH/bin:$PATH"

            # Load .env.local if it exists
            if [ -f .env.local ]; then
              set -a
              source .env.local
              set +a
            fi
          '';
        };
      }
    );
}
```

Enter the dev shell:

```bash
nix develop
# Or with direnv auto-activation:
echo "use flake" > .envrc
direnv allow
```

---

## Option 3: Makefile Bootstrap (Universal)

For teams where Nix and Docker are too opinionated, a `Makefile` with a `setup` target provides a documented, repeatable setup that works anywhere:

```makefile
# Makefile
.DEFAULT_GOAL := help
SHELL := /bin/bash

# Tool versions
GO_VERSION := 1.22.3
NODE_VERSION := 20
TERRAFORM_VERSION := 1.8.5

.PHONY: help
help: ## Show this help
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)

.PHONY: setup
setup: check-deps install-tools setup-hooks setup-env ## Full dev environment setup
	@echo "Dev environment ready"

.PHONY: check-deps
check-deps: ## Check required system dependencies
	@command -v git  >/dev/null || (echo "ERROR: git not found" && exit 1)
	@command -v docker >/dev/null || (echo "ERROR: docker not found" && exit 1)
	@command -v make >/dev/null || (echo "ERROR: make not found" && exit 1)
	@echo "System dependencies OK"

.PHONY: install-tools
install-tools: ## Install project-specific tools
	@./scripts/install-tools.sh

.PHONY: setup-hooks
setup-hooks: ## Install git hooks
	@which pre-commit > /dev/null || pip install pre-commit
	pre-commit install
	pre-commit install --hook-type commit-msg
	@echo "Git hooks installed"

.PHONY: setup-env
setup-env: ## Set up local environment file
	@if [ ! -f .env.local ]; then \
		cp .env.example .env.local; \
		echo ".env.local created from .env.example — fill in your values"; \
	else \
		echo ".env.local already exists"; \
	fi

.PHONY: dev
dev: ## Start development services
	docker-compose -f docker-compose.dev.yml up -d
	air  # Or: npm run dev, etc.

.PHONY: test
test: ## Run tests
	go test ./...

.PHONY: clean
clean: ## Stop and clean development services
	docker-compose -f docker-compose.dev.yml down -v
```

```bash
#!/bin/bash
# scripts/install-tools.sh
set -euo pipefail

install_go() {
  if go version 2>/dev/null | grep -q "go${GO_VERSION:-1.22}"; then
    echo "Go already installed: $(go version)"
    return
  fi
  OS=$(uname -s | tr '[:upper:]' '[:lower:]')
  ARCH=$(uname -m | sed 's/x86_64/amd64/' | sed 's/aarch64/arm64/')
  curl -fsSL "https://go.dev/dl/go${GO_VERSION:-1.22.3}.${OS}-${ARCH}.tar.gz" \
    | sudo tar -C /usr/local -xz
  echo "Go ${GO_VERSION:-1.22.3} installed"
}

install_golangci_lint() {
  curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
    | sh -s -- -b "$(go env GOPATH)/bin" latest
}

install_go
install_golangci_lint
```

---

## Documenting the Template

Every repo should have an `ONBOARDING.md` that fits on one screen:

```markdown
# Getting Started

## Prerequisites
- macOS 13+ / Ubuntu 22.04+ / Windows 11 + WSL2
- Docker Desktop 4.x
- VS Code with Dev Containers extension (recommended)
- Git 2.40+

## Setup (5 minutes)

```bash
git clone git@github.com:your-org/your-repo.git
cd your-repo

# Option A: Dev Container (recommended)
code . # VS Code prompts to reopen in container

# Option B: Local setup
make setup
```

## Running the Project

```bash
make dev # Start all services
open http://localhost:3000
```

## Environment Variables
Copy `.env.example` to `.env.local` and fill in values. Ask in #dev-setup for secrets.
```

---

## Related Reading

- [Remote Team Git Hooks Standardization Guide](/remote-work-tools/remote-team-git-hooks-standardization-guide/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [How to Set Up Woodpecker CI for Self-Hosted](/remote-work-tools/how-to-set-up-woodpecker-ci-for-self-hosted/)

- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)
---

## Related Articles

- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
- [Remote Team Environment Provisioning Tool for Spinning Up](/remote-work-tools/remote-team-environment-provisioning-tool-for-spinning-up-de/)
- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)
- [Setting Up a Remote Dev Server with Hetzner](/remote-work-tools/setting-up-remote-dev-server-with-hetzner/)
- [VS Code Remote Development Setup Guide](/remote-work-tools/vscode-remote-development-setup/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
