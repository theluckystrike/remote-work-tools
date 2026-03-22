---
layout: default
title: "Setting Up a Remote Dev Server with Hetzner"
description: "Provision a Hetzner Cloud dev server with code-server, Tailscale, and automated snapshots — a complete setup for remote development with VS Code in the browser"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-remote-dev-server-with-hetzner/
categories: [guides]
tags: [remote-work-tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Hetzner offers the best price-to-performance ratio for cloud dev servers in Europe and the US. A CX22 (2 vCPU, 4GB RAM) costs €3.79/month. A CCX33 (8 dedicated vCPU, 32GB RAM) costs €27.49/month. Compare that to AWS or GCP equivalents at 3-5x the price. For remote developers who want a persistent, fast dev environment accessible from any machine, Hetzner plus code-server is hard to beat.

## Architecture

```
Your machine (thin client)
    ↓ (browser or VS Code Remote-SSH)
Tailscale VPN
    ↓
Hetzner CX22/CX32 server (code-server + your dev tools)
    ↓
Your projects (on-server, backed up to Hetzner Object Storage)
```

## Step 1: Create the Server

```bash
# Install hcloud CLI
brew install hcloud  # macOS
# or: curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz | tar xz

# Authenticate
hcloud context create myproject
# Enter your API token from Hetzner Cloud Console → Security → API Tokens

# Create SSH key
ssh-keygen -t ed25519 -C "dev-server" -f ~/.ssh/hetzner_dev
hcloud ssh-key create --name dev-server --public-key-file ~/.ssh/hetzner_dev.pub

# Create the server
hcloud server create \
  --name dev-server \
  --type cx32 \
  --image ubuntu-24.04 \
  --ssh-key dev-server \
  --location nbg1 \
  --user-data-from-file cloud-init.yaml

# Get the IP
hcloud server ip dev-server
```

## Step 2: Cloud-Init Configuration

```yaml
# cloud-init.yaml
#cloud-config
users:
  - name: dev
    groups: sudo, docker
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-public-key

packages:
  - curl
  - git
  - htop
  - tmux
  - unzip
  - build-essential
  - docker.io
  - docker-compose-plugin

package_update: true
package_upgrade: true

runcmd:
  # Enable docker without sudo for dev user
  - usermod -aG docker dev
  # Install mise (multi-tool version manager)
  - curl https://mise.run | sh
  # Install Tailscale
  - curl -fsSL https://tailscale.com/install.sh | sh
  # Install code-server
  - curl -fsSL https://code-server.dev/install.sh | sh
  # Enable code-server as systemd service
  - systemctl enable --now code-server@dev
```

## Step 3: Install and Configure code-server

```bash
# SSH into the server
ssh -i ~/.ssh/hetzner_dev dev@<server-ip>

# code-server config is at ~/.config/code-server/config.yaml
cat > ~/.config/code-server/config.yaml << 'EOF'
bind-addr: 127.0.0.1:8080
auth: password
password: $(openssl rand -hex 24)
cert: false
EOF

# Set a strong password
sed -i "s/\$(openssl rand -hex 24)/$(openssl rand -hex 24)/" ~/.config/code-server/config.yaml

sudo systemctl restart code-server@dev
```

## Step 4: Tailscale for Secure Access

Tailscale creates a private network between your devices without exposing the server to the internet.

```bash
# On the Hetzner server
sudo tailscale up --ssh

# On your local machine
tailscale up  # if not already running

# Now access code-server through Tailscale SSH tunnel
ssh -i ~/.ssh/hetzner_dev -L 8080:localhost:8080 dev@100.x.x.x

# Open http://localhost:8080 in your browser
```

**Better: use VS Code Remote-SSH directly:**

```json
// ~/.ssh/config
Host hetzner-dev
    HostName 100.x.x.x  # Tailscale IP
    User dev
    IdentityFile ~/.ssh/hetzner_dev
    ServerAliveInterval 60
```

Then in VS Code: Remote-SSH → Connect to Host → hetzner-dev

## Step 5: Dev Environment Setup with mise

```bash
# SSH into the server as dev user
# Install common tools via mise
eval "$(~/.local/bin/mise activate bash)"

mise use --global node@lts
mise use --global python@3.12
mise use --global go@1.23
mise use --global rust@stable

# Verify
node --version && python --version && go version
```

**Project-specific versions via `.mise.toml`:**

```toml
# .mise.toml (in project root)
[tools]
node = "20.11"
python = "3.12"

[env]
DATABASE_URL = "postgresql://localhost:5432/myapp_dev"
```

## Step 6: Persistent Docker Services

```yaml
# ~/docker-compose.yml — persistent dev services
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"

  mailhog:
    image: mailhog/mailhog
    restart: unless-stopped
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI

volumes:
  postgres_data:
```

```bash
cd ~
docker compose up -d
# Services start automatically on server reboot
```

## Step 7: Automated Snapshots

```bash
# snapshot.sh — run via cron
#!/bin/bash
SERVER_ID=$(hcloud server describe dev-server -o json | jq -r '.id')
SNAPSHOT_NAME="dev-server-$(date +%Y%m%d-%H%M)"

hcloud server create-image \
  --name "$SNAPSHOT_NAME" \
  --type snapshot \
  "$SERVER_ID"

# Keep only the last 5 snapshots
hcloud image list --type snapshot -o json | \
  jq -r 'sort_by(.created) | .[:-5] | .[].id' | \
  xargs -I{} hcloud image delete {}

echo "Snapshot $SNAPSHOT_NAME created"
```

```bash
# Add to crontab (runs every Sunday at 2am)
(crontab -l; echo "0 2 * * 0 HCLOUD_TOKEN=your-token /home/dev/snapshot.sh >> /home/dev/snapshot.log 2>&1") | crontab -
```

## Step 8: Dotfiles Sync

```bash
# Use chezmoi for dotfiles management
sh -c "$(curl -fsLS get.chezmoi.io)"

# Initialize from your dotfiles repo
chezmoi init https://github.com/yourusername/dotfiles.git
chezmoi apply

# On any new server: two commands and you're configured
```

## Cost Calculation

| Server Type | vCPU | RAM | Price/month | Good for |
|---|---|---|---|---|
| CX22 | 2 shared | 4GB | €3.79 | Light work, scripts |
| CX32 | 4 shared | 8GB | €5.39 | Typical dev work |
| CCX23 | 4 dedicated | 8GB | €13.99 | Compile-heavy work |
| CCX33 | 8 dedicated | 32GB | €27.49 | Multiple services, Docker |

An extra €80/year in snapshots gets you weekly backups. Total for a solid dev server: €7-35/month depending on size.

## Firewalla for Hetzner Firewall (Optional)

Lock down the server via Hetzner's cloud firewall:

```bash
# Allow only Tailscale and SSH from specific IPs
hcloud firewall create --name dev-firewall

# Allow SSH only from your home IP
hcloud firewall add-rule dev-firewall \
  --direction in \
  --protocol tcp \
  --port 22 \
  --source-ips "your.home.ip/32"

# Apply to server
hcloud firewall apply-to-resource dev-firewall \
  --type server \
  --server dev-server
```

With Tailscale, you can block port 22 entirely and use only Tailscale SSH.

## Related Reading

- [Portable Dev Environment with Docker 2026](/portable-dev-environment-docker-2026/)
- [Best Remote Dev Server Setup for Async Teams](/best-deploy-workflow-for-a-remote-infrastructure-team-of-3/)
- [Remote Team Deployment Pipeline Best Practices](/how-to-secure-remote-team-ci-cd-pipeline-from-supply-chain-a/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
