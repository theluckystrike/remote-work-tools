---
layout: default
title: "How to Automate Dev Environment Setup: A Practical Guide"
description: "A hands-on guide to automating your development environment setup. Learn to use configuration management tools, shell scripts, and containerization to."
date: 2026-03-15
author: "Remote Work Tools Guide"
permalink: /how-to-automate-dev-environment-setup/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
---
{% raw %}


# How to Automate Dev Environment Setup: A Practical Guide

Automate your dev environment setup by writing shell scripts for package installation, using Docker to containerize your runtime, and layering Ansible playbooks for team-wide configuration management. Store all setup logic in version control so every machine converges on an identical, reproducible state in minutes instead of hours.

This guide walks through each approach with copy-paste examples you can adapt immediately, from a basic bash setup script to a full docker-compose stack and an Ansible playbook.

## Why Automate Your Development Environment

Manual setup processes are error-prone and difficult to reproduce. When you configure a machine by hand, you accumulate subtle differences over time—specific package versions, custom shell aliases, project-specific tools—that never get documented. Team members joining a project inherit inconsistent setups, leading to "works on my machine" bugs and wasted debugging time.

Automation solves these problems by codifying your environment as version-controlled configuration. When your setup lives in code, you can review changes through pull requests, roll back problematic updates, and apply identical configurations across any number of machines. New team members can go from zero to a fully configured development environment in minutes rather than days.

## Starting Simple: Shell Scripts

The most accessible approach to environment automation uses shell scripts. Even basic bash scripts that automate package installation significantly reduce setup time and ensure consistency.

A straightforward setup script might look like this:

```bash
#!/bin/bash
set -e

# Update package lists
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git curl wget vim unzip zsh

# Install programming language runtimes
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Configure git
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Install Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended
```

This script handles the initial machine configuration. Run it on a fresh Ubuntu machine and you get a consistent baseline setup in minutes. The script is version-controllable—you can track changes, branch for different OS variants, and review modifications before applying them.

For project-specific dependencies, create scripts within each repository:

```bash
#!/bin/bash
# project-setup.sh - Run from project root

# Install Node.js dependencies
npm install

# Copy environment configuration
cp .env.example .env

# Initialize pre-commit hooks
npm run prepare

echo "Project environment ready. Run 'npm run dev' to start."
```

## Using Docker for Reproducible Environments

Docker provides stronger isolation than shell scripts by containerizing your entire development environment. This approach packages not just dependencies but the exact runtime environment, eliminating OS-level differences entirely.

A basic development Dockerfile:

```dockerfile
FROM node:20-slim

WORKDIR /app

# Install development dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy package files first for better caching
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY . .

# Expose development server port
EXPOSE 3000

CMD ["npm", "run", "dev"]
```

For local development, use docker-compose to orchestrate multi-service environments:

```yaml
version: '3.8'
services:
  app:
    build: .
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: developer
      POSTGRES_PASSWORD: devpass
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

With this configuration, any team member runs `docker-compose up` and gets a complete development stack—application, database, and cache—without installing anything beyond Docker.

## Configuration Management with Ansible

For more complex environments across multiple machines, Ansible provides automation at scale. Ansible uses declarative YAML files called playbooks to describe desired system states, handling the complexity of idempotent configuration automatically.

An Ansible playbook for development machine setup:

```yaml
---
- name: Configure development workstation
  hosts: localhost
  become: yes
  vars:
    developer_username: developer
    programming_languages:
      - { name: nodejs, version: "20" }
      - { name: python, version: "3.11" }

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install system packages
      apt:
        name:
          - git
          - curl
          - wget
          - vim
          - unzip
          - build-essential
          - docker.io
        state: present

    - name: Configure git global settings
      git_config:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        scope: global
      loop:
        - { name: "user.name", value: "Developer" }
        - { name: "user.email", value: "dev@example.com" }
        - { name: "init.defaultBranch", value: "main" }

    - name: Install pyenv for Python management
      become_user: "{{ developer_username }}"
      shell: |
        curl https://pyenv.run | bash
      args:
        creates: "/home/{{ developer_username }}/.pyenv"

    - name: Install VS Code extensions
      community.general.vscode_extension:
        executable: code
        extension_ids:
          - ms-python.python
          - esbenp.prettier-vscode
          - github.copilot
```

Run this playbook with `ansible-playbook development.yml` and Ansible ensures your machine matches the specification. The idempotent nature means running the playbook multiple times produces the same result—safe for repeated application or CI/CD pipelines.

## Dotfiles: Personal Configuration Management

Beyond project-specific tools, developers accumulate personal configuration through dotfiles—hidden configuration files like `.bashrc`, `.zshrc`, `.vimrc`, and `.gitconfig`. Managing these as a dotfiles repository provides portable personal environments.

A minimal dotfiles structure:

```
dotfiles/
├── .gitignore
├── install.sh
├── .zshrc
├── .vimrc
├── .gitconfig
└── .config/
    └── starship.toml
```

The installation script symlinks files to their expected locations:

```bash
#!/bin/bash
DOTFILES_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

link_file() {
  local source="$DOTFILES_DIR/$1"
  local target="$HOME/$1"
  
  if [ -e "$target" ]; then
    if [ -L "$target" ]; then
      echo "Skipping $1 (already linked)"
    else
      echo "Backing up $1"
      mv "$target" "$target.backup"
    fi
  fi
  
  mkdir -p "$(dirname "$target")"
  ln -sf "$source" "$target"
  echo "Linked $1"
}

link_file ".zshrc"
link_file ".vimrc"
link_file ".gitconfig"
link_file ".config/starship.toml"
```

Combine this with a shell script that installs dependencies, and you have a complete personal environment reproducible across any machine.

## Automating for Teams

Team environments benefit most from automation because they multiply the effort saved across multiple developers. Consider storing setup scripts in a dedicated repository accessible to all team members. Use tools like Machete or GitHub's Template Repositories to provide standardized starting points for new projects.

Documentation matters as much as the scripts themselves. Include README files explaining how to run setup scripts, what assumptions the automation makes about the base system, and how to troubleshoot common issues. Even the best automation fails when users don't understand how to use it or what went wrong when something breaks.

Start with shell scripts, add Docker for project reproducibility, and layer Ansible for team-wide infrastructure management as your needs grow.


## Related Reading

- More guides coming soon.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
