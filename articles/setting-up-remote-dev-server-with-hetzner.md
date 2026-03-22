---
layout: default
title: "Setting Up a Remote Dev Server with Hetzner"
description: "Provision a Hetzner Cloud dev server with code-server, Tailscale, and automated snapshots — a complete setup for remote development with VS Code in the browser"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-remote-dev-server-with-hetzner/
categories: [guides]
tags: [remote-work-tools, remote-work]
reviewed: true
score: 6
intent-checked: true
voice-checked: true
---

{% raw %}

Hetzner offers the best price-to-performance ratio for cloud dev servers in Europe and the US. A CX22 (2 vCPU, 4GB RAM) costs €3.79/month. A CCX33 (8 dedicated vCPU, 32GB RAM) costs €27.49/month. Compare that to AWS or GCP equivalents at 3-5x the price. For remote developers who want a persistent, fast dev environment accessible from any machine, Hetzner plus code-server is hard to beat.

The core idea is simple: instead of lugging a powerful laptop everywhere, or trying to sync dev environments across multiple machines, you run everything on a single cloud server. Your local machine becomes a thin client. Any laptop — even a base MacBook Air or a cheap Chromebook — can be your full workstation via browser or SSH.

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

This architecture has meaningful advantages over local development. The server is always on — long-running jobs, build caches, and Docker services persist between sessions. You get consistent performance regardless of where you're working. And because the server is on Hetzner's network, git operations, Docker pulls, and package downloads are significantly faster than on a home connection.

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

**Location selection**: Hetzner operates datacenters in Nuremberg (nbg1), Falkenstein (fsn1), Helsinki (hel1), and Ashburn VA (ash). Pick the one closest to your primary clients or CI systems, not closest to you — latency to the server over SSH is negligible; latency between your server and external services matters more.

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

Cloud-init runs on first boot and gives you a fully configured server in about 3-4 minutes. No manual SSH steps, no post-boot scripts to remember. The entire configuration is in version control.

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

code-server gives you VS Code in the browser — full extension support, integrated terminal, git integration. For extensions that don't work well in the browser (debuggers for some languages, for example), Remote-SSH is the better option and is covered in Step 4.

One important note: bind code-server to 127.0.0.1, not 0.0.0.0. You never want code-server exposed directly to the internet. Access it only through the Tailscale tunnel.

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

With `--ssh` flag, Tailscale manages SSH keys automatically. You can remove the Hetzner server from all public networks and rely entirely on Tailscale for access. This is the recommended setup: no public SSH port, no exposed services, no firewall rules to maintain for your own access.

## Step 5: Dev Environment Setup with mise

mise (formerly rtx) is a unified tool version manager that replaces nvm, rbenv, pyenv, and goenv with a single tool. It reads `.mise.toml` files in project directories and switches versions automatically.

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

When you `cd` into a project directory, mise reads the `.mise.toml` and activates the correct toolchain. No more "works on my machine" issues from version mismatches across team members — everyone uses the same `.mise.toml` in the repository.

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

The `restart: unless-stopped` policy means your Postgres and Redis instances are always running when the server is. No more "wait for the database to start" in your morning routine, and no state loss between sessions. Mailhog captures all outgoing email from your dev environment so you can test transactional emails without a real SMTP server.

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

Hetzner charges €0.0119 per GB per month for snapshots. A typical dev server snapshot is 8-15GB, so keeping five weekly snapshots costs roughly €0.50-0.90/month. That is a trivial insurance cost against accidental data loss or a corrupted server.

For project data specifically, also consider Hetzner Object Storage (S3-compatible, €4.43/month for 1TB) combined with `restic` or `rclone` for automated project backup. This gives you independent recovery for your code and databases even if the server itself needs to be rebuilt from scratch.

## Step 8: Dotfiles Sync

```bash
# Use chezmoi for dotfiles management
sh -c "$(curl -fsLS get.chezmoi.io)"

# Initialize from your dotfiles repo
chezmoi init https://github.com/yourusername/dotfiles.git
chezmoi apply

# On any new server: two commands and you're configured
```

chezmoi is preferable to bare git for dotfiles because it handles machine-specific values (different SSH keys, different email addresses per machine) cleanly via templates. Your `.zshrc`, `.gitconfig`, shell aliases, and tmux config are all in one place and applied consistently across every machine you use.

## Cost Calculation

| Server Type | vCPU | RAM | Price/month | Good for |
|---|---|---|---|---|
| CX22 | 2 shared | 4GB | €3.79 | Light work, scripts |
| CX32 | 4 shared | 8GB | €5.39 | Typical dev work |
| CCX23 | 4 dedicated | 8GB | €13.99 | Compile-heavy work |
| CCX33 | 8 dedicated | 32GB | €27.49 | Multiple services, Docker |

An extra €80/year in snapshots gets you weekly backups. Total for a solid dev server: €7-35/month depending on size. Compare to GitHub Codespaces at $0.18/hour for a 4-core instance — that's $130/month if you work 720 hours. Hetzner at €5.39 for the same class of machine is a 24x cost reduction.

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

With Tailscale, you can block port 22 entirely and use only Tailscale SSH. The recommended final state is: no public-facing ports at all except UDP 41641 for Tailscale's WireGuard traffic. Everything else routes through the Tailscale mesh.

## Comparing Hetzner to Alternatives

| Provider | 4-core / 8GB RAM | Monthly cost | Notes |
|---|---|---|---|
| Hetzner CX32 | 4 shared vCPU, 8GB | €5.39 | Best value for shared CPU |
| Hetzner CCX23 | 4 dedicated vCPU, 8GB | €13.99 | True dedicated cores |
| DigitalOcean 4GB | 2 vCPU, 4GB | $24 | ~3x more expensive |
| AWS EC2 t3.large | 2 vCPU, 8GB | ~$60 | ~10x more expensive |
| GitHub Codespaces | 4-core | ~$130/mo (720h) | Per-hour billing adds up |
| Fly.io (performance-2x) | 2 vCPU, 4GB | $62 | Good if you need edge locations |

For developers in Europe or with European client bases, Hetzner is the clear default. For developers who need a US presence, Hetzner Ashburn (ash) gives the same economics in Virginia.

## Related Reading

- [Portable Dev Environment with Docker 2026](/portable-dev-environment-docker-2026/)
- [Best Remote Dev Server Setup for Async Teams](/best-deploy-workflow-for-a-remote-infrastructure-team-of-3/)
- [Remote Team Deployment Pipeline Best Practices](/how-to-secure-remote-team-ci-cd-pipeline-from-supply-chain-a/)
---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
