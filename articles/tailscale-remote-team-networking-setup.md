---
layout: default
title: "Tailscale for Remote Team Networking Setup"
description: "Set up Tailscale for remote team networking: install on all devices, configure ACLs, set up subnet routes and exit nodes, and replace your VPN with a mesh network."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /tailscale-remote-team-networking-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Tailscale turns every device your team uses into a node on a private network, without requiring a central VPN server, NAT traversal rules, or certificate management. Each device gets a stable IP in the `100.64.0.0/10` range, reachable from any other device on the tailnet regardless of what network either is on.

For remote teams, Tailscale replaces the classic VPN setup with something that works in 10 minutes, handles firewall traversal automatically, and scales to hundreds of devices without extra configuration.

## Install on All Platforms

```bash
# macOS
brew install tailscale
# or download the App Store app (recommended for menu bar integration)

# Linux (Ubuntu/Debian)
curl -fsSL https://tailscale.com/install.sh | sh

# RHEL/CentOS/Amazon Linux
curl -fsSL https://tailscale.com/install.sh | sh

# Raspberry Pi (armv7/arm64)
curl -fsSL https://tailscale.com/install.sh | sh

# Windows
# Download installer from tailscale.com/download

# Docker
docker run -d \
  --name tailscale \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  --device /dev/net/tun \
  -v /var/lib/tailscale:/var/lib/tailscale \
  -e TS_AUTHKEY=tskey-auth-XXXXXX \
  tailscale/tailscale
```

## Authenticate and Start

```bash
# Start Tailscale and open browser to authenticate
sudo tailscale up

# Headless authentication (for servers) — generate auth key in admin console
sudo tailscale up --authkey tskey-auth-XXXXXX

# Check status
tailscale status
# Shows all connected devices with IPs, last seen, and OS

# Ping another device
tailscale ping 100.64.0.5
tailscale ping hostname-of-device

# SSH to a device using Tailscale IP
ssh ubuntu@100.64.0.5

# Or with MagicDNS, use the device hostname directly
ssh ubuntu@my-dev-server
```

## Enable MagicDNS and HTTPS

MagicDNS assigns each device a DNS name like `device-name.tail1234.ts.net`:

1. Go to Tailscale admin console → DNS
2. Enable MagicDNS
3. Enable HTTPS certificates (Let's Encrypt, automated)

After enabling:

```bash
# Access a device by hostname instead of IP
ssh ubuntu@devserver.tail1234.ts.net
curl https://staging-app.tail1234.ts.net

# Get HTTPS cert for a service running on a Tailscale node
tailscale cert staging-app.tail1234.ts.net
# Outputs cert.pem and key.pem in current directory
```

## Configure ACLs (Access Control Lists)

ACLs control which devices can reach which. By default, all devices on a tailnet can reach each other. For teams, restrict access:

```json
// Tailscale ACL policy (JSON with comments)
// Set in admin console → Access Controls

{
  "groups": {
    "group:engineering": ["user:alice@company.com", "user:bob@company.com"],
    "group:devops": ["user:carol@company.com"],
    "group:contractors": ["user:dave@contractco.com"]
  },

  "tagOwners": {
    "tag:prod-server": ["group:devops"],
    "tag:dev-server": ["group:engineering"],
    "tag:exit-node": ["group:devops"]
  },

  "acls": [
    // Engineering can reach dev servers
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:dev-server:*"]
    },
    // DevOps can reach everything
    {
      "action": "accept",
      "src": ["group:devops"],
      "dst": ["*:*"]
    },
    // Contractors can only reach specific services on dev
    {
      "action": "accept",
      "src": ["group:contractors"],
      "dst": ["tag:dev-server:443", "tag:dev-server:80"]
    },
    // Everyone can reach exit nodes
    {
      "action": "accept",
      "src": ["*"],
      "dst": ["tag:exit-node:*"]
    }
  ],

  "ssh": [
    // Engineering can SSH to dev servers
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:dev-server"],
      "users": ["ubuntu", "ec2-user"]
    }
  ]
}
```

Tag a device when authenticating:

```bash
# Tag a server as a dev server during auth
sudo tailscale up --authkey tskey-auth-XXXXXX --advertise-tags tag:dev-server
```

## Set Up Subnet Routes

Subnet routes let Tailscale nodes access private subnets — for example, your AWS VPC or office LAN — without installing Tailscale on every machine in the subnet.

```bash
# On the gateway/subnet router (a Tailscale node in the subnet)
# Enable IP forwarding first
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Advertise the subnet
sudo tailscale up --advertise-routes=10.0.1.0/24,10.0.2.0/24

# In the admin console: approve the advertised routes
# Admin → Machines → select machine → Edit route settings → approve routes

# On client devices: use the route
# By default, clients use approved subnet routes automatically
# Verify:
tailscale status
# Shows subnet routers with the routes they advertise
```

## Exit Nodes

An exit node routes all internet traffic for a device through a Tailscale node — equivalent to a VPN exit point.

```bash
# Set up a node as an exit node
sudo tailscale up --advertise-exit-node

# Approve in admin console: Machines → select → Edit route settings → Use as exit node

# On client: use the exit node
sudo tailscale up --exit-node=100.64.0.5
# or by hostname
sudo tailscale up --exit-node=exit-server

# Disable exit node (go back to direct routing)
sudo tailscale up --exit-node=

# Verify your IP is now the exit node's IP
curl https://ipinfo.io
```

## Tailscale SSH (Replace SSH Key Management)

Tailscale SSH uses your Tailscale identity instead of SSH keys. No more distributing authorized_keys files:

```bash
# Enable Tailscale SSH on a server
sudo tailscale up --ssh

# The server now accepts SSH from authorized Tailscale users
# No ~/.ssh/authorized_keys needed

# SSH from any authorized device
ssh ubuntu@devserver  # uses Tailscale identity, no key needed

# Check Tailscale SSH audit log in admin console
# Admin → Logs → SSH
```

Enable Tailscale SSH in ACL policy:

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:engineering"],
      "dst": ["tag:dev-server"],
      "users": ["autogroup:nonroot", "ubuntu"]
    },
    {
      "action": "check",  // require re-auth for prod
      "src": ["group:devops"],
      "dst": ["tag:prod-server"],
      "users": ["ubuntu"]
    }
  ]
}
```

## Running Tailscale on Servers at Boot

```bash
# systemd service is installed automatically via the install script
# Verify it's enabled
sudo systemctl status tailscaled

# Auto-start with auth key (for automated server provisioning)
sudo systemctl enable tailscaled
sudo tailscale up --authkey tskey-auth-XXXXXX --advertise-tags tag:dev-server

# For ephemeral nodes (e.g., CI runners) that deregister when stopped
sudo tailscale up --authkey tskey-auth-XXXXXX --ephemeral
```



## Related Articles

- [Remote Work VPN for Teams Comparison 2026: Tailscale vs.](/remote-work-tools/remote-work-vpn-for-teams-comparison-2026/)
- [Freelance Developer Networking Strategies Online: A](/remote-work-tools/freelance-developer-networking-strategies-online/)
- [Best Bug Tracking Setup for a 7-Person Remote QA Team](/remote-work-tools/best-bug-tracking-setup-for-a-7-person-remote-qa-team/)
- [Best Two-Factor Authentication Setup for Remote Team Shared](/remote-work-tools/best-two-factor-authentication-setup-for-remote-team-shared-/)
- [Certificate Based Authentication Setup for Remote Team VPN](/remote-work-tools/certificate-based-authentication-setup-for-remote-team-vpn-c/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
