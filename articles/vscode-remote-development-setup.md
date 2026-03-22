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
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

VS Code's Remote Development extensions let you run your editor UI locally while the code, terminal, debugger, and extensions all run on a remote server. You get the performance of a powerful remote machine and the latency of a local editor window.

This guide covers the SSH remote extension, dev containers, settings sync, and per-project configuration that makes remote development practical for teams.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install the Remote Development Extension Pack

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

### Step 2: Remote SSH Setup

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

### Step 3: Dev Containers

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

### Step 4: Settings Sync for Remote Teams

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

### Step 5: Useful Remote Development Settings

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

### Step 6: Port Forwarding

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

### Step 7: Debugging on Remote Hosts

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

## Remote IDE Comparison

VS Code is not the only option for remote development. Understanding the trade-offs helps teams make the right call:

| Tool | Remote protocol | Language support | Container support | Cost |
|------|----------------|-----------------|------------------|------|
| **VS Code Remote SSH** | SSH + VS Code server | Universal (extension-based) | Dev Containers spec | Free |
| **JetBrains Gateway** | SSH + JetBrains backend | Excellent for Java, Kotlin, Python | Yes (via JetBrains Space) | Paid (IDE license required) |
| **GitHub Codespaces** | Browser or VS Code | Universal | Dev Containers spec | Usage-based (~$0.18/hr for 2-core) |
| **Gitpod** | Browser or VS Code/JetBrains | Universal | Workspace images | Free tier, paid from $9/mo |
| **Cursor** | SSH + Cursor server | Universal + AI pair programming | Limited | $20/mo with AI features |

VS Code Remote SSH wins on cost and control for teams with existing infrastructure. GitHub Codespaces or Gitpod make sense when you want zero-config onboarding for contributors who shouldn't need a local dev environment at all.

### Step 8: Step-by-Step: First-Time Remote SSH Setup

**Step 1 — Generate an SSH key pair on your local machine.** Run `ssh-keygen -t ed25519 -C "your.email@example.com"`. Ed25519 keys are smaller and faster than RSA.

**Step 2 — Copy your public key to the remote server.** Run `ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@dev.example.com`. Alternatively, append the contents of `~/.ssh/id_ed25519.pub` to `~/.ssh/authorized_keys` on the remote machine.

**Step 3 — Add the host to `~/.ssh/config`.** Use the format shown in the SSH Config section above. Include `ServerAliveInterval 30` to prevent idle disconnects.

**Step 4 — Test the connection without VS Code first.** Run `ssh devserver` from your terminal. If it connects without a password prompt, VS Code will work.

**Step 5 — Open VS Code and connect.** Press `Ctrl+Shift+P`, type `Remote-SSH: Connect to Host`, select your host. VS Code will install its server component on the remote machine — this takes about 30 seconds the first time.

**Step 6 — Open your project folder.** Use `File → Open Folder` and navigate to your project directory on the remote machine. The path is on the remote filesystem, not local.

**Step 7 — Install project-recommended extensions.** VS Code will prompt you to install the extensions listed in `.vscode/extensions.json`. Accept the prompt. Extensions install on the remote server and run there.

**Step 8 — Commit your `.vscode/` config files.** Committing `settings.json`, `extensions.json`, `launch.json`, and `tasks.json` means every teammate who opens the repo gets a consistent environment automatically.

## Performance Tips for Remote Development

Remote development introduces network latency between your keyboard and the language server. These settings reduce perceived lag:

Disable file watchers for large directories. The `files.watcherExclude` setting prevents VS Code from watching `node_modules` and `dist` directories on the remote machine, which can consume significant CPU on large projects.

Use `remote.SSH.remoteServerListenOnSocket: true` instead of the default TCP port for the VS Code server connection. Unix sockets are faster than TCP localhost connections on the same machine.

Enable persistent terminal sessions. With `terminal.integrated.enablePersistentSessions: true`, your terminal state survives network blips. Long-running processes like `npm run dev` keep going even if your SSH connection briefly drops.

Set a 90-second connection timeout for slow networks. Add `"remote.SSH.connectTimeout": 90` if you regularly connect from high-latency networks or VPNs.

## FAQ

**Why do my locally installed extensions not work on the remote server?**
Most extensions need to run where the code is — on the remote server. UI extensions like themes run locally, but language servers, linters, and debuggers install and run on the remote. Open the Extensions panel while connected and install them explicitly for the remote host.

**Can I use VS Code Remote SSH through a corporate VPN?**
Yes. Add the remote host to your `~/.ssh/config` with the VPN-accessible hostname. If the server is behind a bastion host, use `ProxyJump bastion.corp.example.com` in your SSH config. VS Code passes all traffic through the SSH tunnel.

**How do dev containers differ from Remote SSH?**
Remote SSH connects to an existing server and uses whatever is installed there. Dev containers spin up a fresh Docker container with a precisely defined environment every time. Dev containers are better for reproducibility across teammates; Remote SSH is better when you need access to a specific persistent server with specific data or resources.

**What happens to my terminal if my internet drops?**
With `terminal.integrated.enablePersistentSessions: true`, VS Code reconnects and your terminal session resumes. For long-running processes you cannot afford to lose, use `tmux` or `screen` on the remote server — these survive SSH disconnections regardless of VS Code settings.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Articles

- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
- [Best Practice for Remote Team Code Review Comments](/remote-work-tools/best-practice-for-remote-team-code-review-comments-keeping-f/)
- [Code Review Workflow for a Remote Backend Team](/remote-work-tools/code-review-workflow-for-a-remote-backend-team-of-6-develope/)
- [How to Scale Remote Team Code Review Process](/remote-work-tools/how-to-scale-remote-team-code-review-process-when-engineerin/)
- [Remote Developer Code Review Workflow Tools for Teams](/remote-work-tools/remote-developer-code-review-workflow-tools-for-teams-without-synchronous-overlap/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
