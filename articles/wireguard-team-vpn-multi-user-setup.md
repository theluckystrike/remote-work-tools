---
layout: default
title: "WireGuard Team VPN: Multi-User Setup Guide"
description: "Set up WireGuard as a team VPN with multiple users, split tunneling, and peer management scripts. Covers server config, peer generation, and client setup for"
date: 2026-03-21
author: theluckystrike
permalink: /wireguard-team-vpn-multi-user-setup/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, vpn]---
---
layout: default
title: "WireGuard Team VPN: Multi-User Setup Guide"
description: "Set up WireGuard as a team VPN with multiple users, split tunneling, and peer management scripts. Covers server config, peer generation, and client setup for"
date: 2026-03-21
author: theluckystrike
permalink: /wireguard-team-vpn-multi-user-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, vpn]---

{% raw %}

WireGuard is the fastest, simplest VPN protocol available. Setting it up for a solo developer takes 20 minutes. Setting it up for a team requires managing peer keys, distributing configs, revoking access, and deciding whether to route all traffic or only internal traffic through the tunnel.

This guide covers the complete team setup: server installation, peer key management scripts, client configuration for multiple platforms, and split tunneling to avoid routing all traffic through the VPN.

## Key Takeaways

- **WireGuard is the fastest**: simplest VPN protocol available.
- **Setting it up for**: a team requires managing peer keys, distributing configs, revoking access, and deciding whether to route all traffic or only internal traffic through the tunnel.
- **What are the most**: common mistakes to avoid? The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully.
- **Topics covered**: architecture, server setup, peer management script

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Architecture

```
VPS / Server (wg server)
  IP: 203.0.113.1 (public)
  WireGuard interface: wg0 (10.8.0.1/24)

Team members (peers)
  mike-laptop:   10.8.0.2/32
  sarah-laptop:  10.8.0.3/32
  alex-laptop:   10.8.0.4/32
  ci-runner:     10.8.0.5/32

Access control:
  All peers can reach internal services via 10.8.0.0/24
  Split tunnel: only 10.8.0.0/24 routes through VPN (not all internet traffic)
```

### Step 2: Server Setup

```bash
# Install WireGuard on Ubuntu 22.04
sudo apt update && sudo apt install wireguard -y

# Generate server keys
cd /etc/wireguard
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key

# Get the values
SERVER_PRIVATE=$(sudo cat /etc/wireguard/server_private.key)
SERVER_PUBLIC=$(sudo cat /etc/wireguard/server_public.key)
echo "Server public key: $SERVER_PUBLIC"
```

```bash
# /etc/wireguard/wg0.conf
sudo tee /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $(cat /etc/wireguard/server_private.key)
Address = 10.8.0.1/24
ListenPort = 51820

# Enable IP forwarding for traffic between peers
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Peers will be appended below by the add-peer script
EOF

sudo chmod 600 /etc/wireguard/wg0.conf
```

```bash
# Enable IP forwarding at the OS level
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p

# Enable and start WireGuard
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Verify it's running
sudo wg show
```

### Step 3: Peer Management Script

Managing keys manually gets error-prone at team scale. This script generates peer configs and appends them to the server config:

```bash
#!/bin/bash
# /opt/wireguard/add-peer.sh
# Usage: ./add-peer.sh mike-laptop 10.8.0.2

set -euo pipefail

PEER_NAME="${1:?Usage: $0 <peer-name> <ip>}"
PEER_IP="${2:?Usage: $0 <peer-name> <ip>}"
SERVER_PUBLIC=$(cat /etc/wireguard/server_public.key)
SERVER_ENDPOINT="203.0.113.1:51820"  # Change to your server IP
VPN_SUBNET="10.8.0.0/24"
DNS="1.1.1.1, 1.0.0.1"

PEERS_DIR="/opt/wireguard/peers/${PEER_NAME}"
mkdir -p "${PEERS_DIR}"

# Generate peer keys
wg genkey | tee "${PEERS_DIR}/private.key" | wg pubkey > "${PEERS_DIR}/public.key"
chmod 600 "${PEERS_DIR}/private.key"

PEER_PRIVATE=$(cat "${PEERS_DIR}/private.key")
PEER_PUBLIC=$(cat "${PEERS_DIR}/public.key")

# Append peer to server config
sudo tee -a /etc/wireguard/wg0.conf << EOF

# ${PEER_NAME}
[Peer]
PublicKey = ${PEER_PUBLIC}
AllowedIPs = ${PEER_IP}/32
EOF

# Generate client config
tee "${PEERS_DIR}/wg0.conf" << EOF
[Interface]
PrivateKey = ${PEER_PRIVATE}
Address = ${PEER_IP}/32
DNS = ${DNS}

[Peer]
PublicKey = ${SERVER_PUBLIC}
Endpoint = ${SERVER_ENDPOINT}
# Split tunnel: only route VPN subnet through WireGuard
# Change to 0.0.0.0/0 for full tunnel (all internet through VPN)
AllowedIPs = ${VPN_SUBNET}
PersistentKeepalive = 25
EOF

# Reload WireGuard to pick up the new peer (zero downtime)
sudo wg syncconf wg0 <(sudo wg-quick strip wg0)

echo "Peer '${PEER_NAME}' added with IP ${PEER_IP}"
echo "Client config: ${PEERS_DIR}/wg0.conf"
echo ""
echo "Distribute ${PEERS_DIR}/wg0.conf to the user."
echo "Or generate QR code: qrencode -t ansiutf8 < ${PEERS_DIR}/wg0.conf"
```

```bash
chmod +x /opt/wireguard/add-peer.sh

# Add team members
sudo /opt/wireguard/add-peer.sh mike-laptop 10.8.0.2
sudo /opt/wireguard/add-peer.sh sarah-laptop 10.8.0.3
sudo /opt/wireguard/add-peer.sh alex-laptop 10.8.0.4
sudo /opt/wireguard/add-peer.sh ci-runner 10.8.0.5
```

### Step 4: Revoke a Peer

```bash
#!/bin/bash
# /opt/wireguard/remove-peer.sh
# Usage: ./remove-peer.sh mike-laptop

PEER_NAME="${1:?Usage: $0 <peer-name>}"
PEERS_DIR="/opt/wireguard/peers/${PEER_NAME}"

if [ ! -f "${PEERS_DIR}/public.key" ]; then
  echo "Peer not found: ${PEER_NAME}"
  exit 1
fi

PEER_PUBLIC=$(cat "${PEERS_DIR}/public.key")

# Remove peer from running WireGuard instance (immediate effect)
sudo wg set wg0 peer "${PEER_PUBLIC}" remove

# Remove peer block from config file
# Find and delete the [Peer] block for this peer
sudo python3 << EOF
import re

with open('/etc/wireguard/wg0.conf', 'r') as f:
    content = f.read()

# Remove the peer block (comment + [Peer] section until next blank line or EOF)
pattern = r'\n# ${PEER_NAME}\n\[Peer\].*?(?=\n\n|\Z)'
cleaned = re.sub(pattern, '', content, flags=re.DOTALL)

with open('/etc/wireguard/wg0.conf', 'w') as f:
    f.write(cleaned)
EOF

# Archive keys (don't delete — useful for audit)
sudo mv "${PEERS_DIR}" "${PEERS_DIR}.revoked"

echo "Peer '${PEER_NAME}' removed. Access revoked immediately."
```

### Step 5: Client Setup: macOS

```bash
# Install WireGuard
brew install wireguard-tools

# Or use the macOS App Store WireGuard app for a GUI

# Command line
sudo cp /path/to/wg0.conf /opt/homebrew/etc/wireguard/wg0.conf
sudo wg-quick up wg0

# Disconnect
sudo wg-quick down wg0

# Check status
sudo wg show
```

### Step 6: Client Setup: Linux

```bash
# Install WireGuard
sudo apt install wireguard -y     # Debian/Ubuntu
sudo pacman -S wireguard-tools    # Arch

# Copy config from server
scp serveruser@203.0.113.1:/opt/wireguard/peers/mike-laptop/wg0.conf \
  /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf

# Connect
sudo wg-quick up wg0

# Disconnect
sudo wg-quick down wg0

# Auto-start on boot (optional — only if you want always-on VPN)
sudo systemctl enable wg-quick@wg0
```

### Step 7: Client Setup: Windows

```
1. Download WireGuard installer from wireguard.com
2. Open WireGuard → Add Tunnel → Import tunnel(s) from file
3. Select the .conf file distributed by the server admin
4. Click "Activate" to connect
```

### Step 8: Split Tunnel vs Full Tunnel

The `AllowedIPs` setting in the client config controls what traffic routes through the VPN:

```ini
# Split tunnel — only VPN subnet goes through WireGuard
# All other internet traffic goes direct (not through the server)
AllowedIPs = 10.8.0.0/24

# Full tunnel — ALL traffic routes through VPN server
# Internet traffic exits through the VPS's public IP
AllowedIPs = 0.0.0.0/0, ::/0
```

**Use split tunnel** when:
- Team members only need to reach internal services (staging, databases, admin panels)
- You do not want to create a bandwidth bottleneck at the VPS
- Team members have privacy concerns about routing all browsing through company infra

**Use full tunnel** when:
- Geographic access restrictions need to be bypassed
- All outbound traffic must originate from a fixed IP (client requirements, third-party services)
- Security policy requires all traffic to be inspected

### Step 9: Monitor Active Connections

```bash
# On the server: show connected peers and last handshake
sudo wg show

# Sample output:
# peer: abc123... (mike-laptop)
#   endpoint: 198.51.100.5:45678
#   allowed ips: 10.8.0.2/32
#   latest handshake: 42 seconds ago
#   transfer: 1.23 MiB received, 2.34 MiB sent

# List all peers with names (from comments in wg0.conf)
grep -A1 "# " /etc/wireguard/wg0.conf | grep -E "# |PublicKey"
```

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [How to Set Up WireGuard VPN Server for Small Remote Development Teams](/remote-work-tools/how-to-set-up-wireguard-vpn-server-for-small-remote-developm/)
- [Best VPN for Remote Development Teams with Split Tunneling](/remote-work-tools/best-vpn-for-remote-development-teams-with-split-tunneling-2/)
- [Home Lab Setup Guide for Remote Developers](/remote-work-tools/home-lab-setup-guide-remote-developers/)
- [How to Set Up WireGuard VPN on iPhone for Always-On Privacy](https://theluckystrike.github.io/privacy-tools-guide/how-to-set-up-wireguard-vpn-on-iphone-for-always-on-privacy-/)
- [WireGuard vs OpenVPN Speed Difference on Mobile Data](https://theluckystrike.github.io/privacy-tools-guide/wireguard-vs-openvpn-speed-difference-on-mobile-data-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to guide?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Is this approach secure enough for production?**

The patterns shown here follow standard practices, but production deployments need additional hardening. Add rate limiting, input validation, proper secret management, and monitoring before going live. Consider a security review if your application handles sensitive user data.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}
