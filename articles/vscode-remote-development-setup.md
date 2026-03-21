---
layout: default
title: "VS Code Remote Development Setup Guide"
description: "Set up VS Code for remote development over SSH, in containers, and with WSL. Extension configs, settings sync, and dev container workflow for distributed teams"
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /vscode-remote-development-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

VS Code's Remote Development extensions let you run your editor UI locally while the code, terminal, debugger, and extensions all run on a remote server. You get the performance of a powerful remote machine and the latency of a local editor window.

This guide covers the SSH remote extension, dev containers, settings sync, and per-project configuration that makes remote development practical for teams.

## Install the Remote Development Extension Pack

```bash
# Install via CLI
code --install-extension ms-vscode-remote.vscode-remote-extensionpack

# This installs three extensions:
# - Remote - SSH (ms-vscode-remote.remote-ssh)
# - Remote - Containers (ms-vscode-remote.remote-containers)
# - Remote - WSL (ms-vscode-remote.remote-wsl)

# Verify installation
code --list-extensions | grep remote
```

Alternatively, open VS Code, press `Ctrl+Shift+X`, and search for "Remote Development".

## Remote SSH Setup

### Configure SSH Access

The Remote SSH extension reads from your `~/.ssh/config`. Set up your hosts there:

```bash
# ~/.ssh/config

Host devserver
    HostName dev.example.com
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
    ServerAliveInterval 30
    ServerAliveCountMax 3

Host jump-dev
    HostName internal.dev.example.com
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion.example.com
    ServerAliveInterval 30
```

`ForwardAgent yes` lets you use your local SSH keys on the remote server (for git operations). `ServerAliveInterval` prevents the connection from dropping on idle.

### Connect and Configure

1. Open the Command Palette (`Ctrl+Shift+P`)
2. Type `Remote-SSH: Connect to Host`
3. Select your host from `~/.ssh/config`

VS Code installs a small server on the remote machine the first time. After that, connections are fast.

### Install Extensions on the Remote Host

Extensions run either locally (UI extensions like themes) or on the remote server (language servers, linters). Install server-side extensions through the extensions panel while connected — they install on the remote machine, not your local one.

```json
// .vscode/extensions.json — recommend extensions for this project
// Teammates get prompted to install these when they open the folder
{
  "recommendations": [
    "ms-python.python",
    "ms-python.vscode-pylance",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "eamodio.gitlens",
    "ms-azuretools.vscode-docker"
  ]
}
```

## Dev Containers

Dev containers define the full development environment in a `.devcontainer/devcontainer.json` file. Everyone on the team gets the same toolchain, runtimes, and extensions — no "works on my machine" issues.

### Basic devcontainer.json

```json
// .devcontainer/devcontainer.json
{
  "name": "Node.js Dev Container",
  "image": "mcr.microsoft.com/devcontainers/node:20",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "customizations": {
    "vscode": {
      "settings": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "eslint.validate": ["javascript", "typescript"]
      },
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "eamodio.gitlens"
      ]
    }
  },
  "forwardPorts": [3000, 5432],
  "postCreateCommand": "npm install",
  "remoteUser": "node",
  "mounts": [
    "source=${localWorkspaceFolder}/.env.local,target=/workspaces/${localWorkspaceFolderBasename}/.env,type=bind,consistency=cached"
  ]
}
```

For a Python project with Docker Compose:

```yaml
# .devcontainer/docker-compose.yml
version: '3.8'
services:
  app:
    build:
      context: ..
      dockerfile: .devcontainer/Dockerfile
    volumes:
      - ..:/workspaces/myapp:cached
      - /var/run/docker.sock:/var/run/docker.sock
    command: sleep infinity
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

```json
// .devcontainer/devcontainer.json (with compose)
{
  "name": "Python + Postgres",
  "dockerComposeFile": "docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/workspaces/myapp",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "ms-toolsai.jupyter"
      ]
    }
  },
  "postCreateCommand": "pip install -r requirements-dev.txt"
}
```

## Settings Sync for Remote Teams

Settings Sync keeps your VS Code configuration consistent across machines. Enable it:

1. Open `Ctrl+Shift+P` → `Settings Sync: Turn On`
2. Sign in with GitHub or Microsoft account
3. Choose what to sync: Settings, Keybindings, Snippets, Extensions, UI State

For teams, use a shared `settings.json` committed to the repo instead of relying on personal sync:

```json
// .vscode/settings.json (commit this to version control)
{
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.formatOnSave": true,
  "editor.rulers": [80, 120],
  "editor.bracketPairColorization.enabled": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.exclude": {
    "**/.git": true,
    "**/node_modules": true,
    "**/__pycache__": true,
    "**/.pytest_cache": true
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.next": true
  },
  "terminal.integrated.defaultProfile.linux": "bash",
  "git.autofetch": true,
  "git.confirmSync": false
}
```

## Useful Remote Development Settings

Add to your user `settings.json` (`Ctrl+Shift+P` → `Open User Settings JSON`):

```json
{
  // SSH remote settings
  "remote.SSH.remoteServerListenOnSocket": true,
  "remote.SSH.connectTimeout": 30,
  "remote.SSH.maxReconnectionAttempts": 5,

  // Performance on remote
  "files.watcherExclude": {
    "**/.git/objects/**": true,
    "**/node_modules/**": true,
    "**/dist/**": true
  },

  // Keep terminal alive on disconnect
  "terminal.integrated.persistentSessionReviveProcess": "onExitAndWindowClose",
  "terminal.integrated.enablePersistentSessions": true,

  // Port forwarding
  "remote.autoForwardPorts": true,
  "remote.autoForwardPortsSource": "process",

  // Show remote indicator in status bar
  "remote.extensionKind": {
    "ms-vscode.cpptools": ["workspace"]
  }
}
```

## Port Forwarding

VS Code automatically detects ports your remote process opens and offers to forward them. You can also set up port forwarding manually:

1. Open the Ports panel: `Ctrl+Shift+P` → `Ports: Focus on Ports View`
2. Click `Forward a Port` and enter the port number
3. The forwarded URL appears in the panel — click to open in browser

For persistent forwarding in `.devcontainer/devcontainer.json`, use `forwardPorts`. For SSH remotes, add to your task configuration:

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Start dev server",
      "type": "shell",
      "command": "npm run dev",
      "isBackground": true,
      "problemMatcher": {
        "owner": "custom",
        "pattern": { "regexp": "." },
        "background": {
          "activeOnStart": true,
          "beginsPattern": "starting",
          "endsPattern": "ready on"
        }
      }
    }
  ]
}
```

## Debugging on Remote Hosts

Launch configurations work the same whether local or remote. The debug adapter runs on the remote machine:

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Node: Remote Debug",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "/app",
      "restart": true,
      "sourceMaps": true
    },
    {
      "name": "Python: Remote",
      "type": "debugpy",
      "request": "launch",
      "program": "${workspaceFolder}/main.py",
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```



## Related Articles

- [Node.js and npm](/remote-work-tools/claude-code-npm-package-development-guide/)
- [Best Practice for Remote Team Code Review Comments](/remote-work-tools/best-practice-for-remote-team-code-review-comments-keeping-f/)
- [Review assignment logic (example)](/remote-work-tools/code-review-workflow-for-a-remote-backend-team-of-6-develope/)
- [Code Review Guidelines](/remote-work-tools/how-to-scale-remote-team-code-review-process-when-engineerin/)
- [Remote Developer Code Review Workflow Tools for Teams](/remote-work-tools/remote-developer-code-review-workflow-tools-for-teams-without-synchronous-overlap/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
