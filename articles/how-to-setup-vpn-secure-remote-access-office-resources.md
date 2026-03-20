---
layout: default
title: "How to Set Up a VPN for Secure Remote Access to Office Resources"
description: "Step-by-step guide to VPN setup for secure remote work. Learn WireGuard vs OpenVPN, firewall configuration, and troubleshooting for teams of any size."
date: 2026-03-20
author: theluckystrike
permalink: /how-to-setup-vpn-secure-remote-access-office-resources/
categories: [guides]
tags: [remote-work, security, vpn]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

VPN security should be transparent—remote workers shouldn't notice it's running. Yet setting up a reliable corporate VPN frustrates IT teams because of configuration complexity, split-brain DNS issues, and client compatibility across Windows, Mac, and Linux. This guide provides production-ready VPN setups for teams of 5-500 people, with specific configuration fixes for common failure modes.

## VPN Architecture: What You Actually Need

A corporate VPN needs:

1. **VPN Gateway** — Central server accepting remote connections
2. **Authentication** — Ensuring only employees connect
3. **Encryption** — Protecting traffic from eavesdropping
4. **DNS Management** — Routing internal.company.com to correct internal IP
5. **Split Tunneling** (Optional) — Allowing local internet traffic without VPN

Most remote workers need access to:
- Internal wikis and documentation
- Code repositories (GitLab, GitHub Enterprise)
- Internal APIs and development environments
- Database query tools
- Project management systems (Jira, Linear)
- Internal collaboration tools

They do NOT need the VPN for:
- Public websites (Gmail, AWS console, GitHub.com)
- Cloud services (Slack, Figma, Datadog)
- Video calls (Zoom, Google Meet)

This guide focuses on **split-tunnel VPN** (routes internal traffic through VPN, public internet directly) rather than full-tunnel (all traffic through VPN).

## VPN Protocol Comparison

| Protocol | Speed | Setup Complexity | Client Support | Best For | Security |
|----------|-------|-----------------|-----------------|----------|----------|
| **WireGuard** | 900+ Mbps | Simple (5 min) | Linux, Mac, iOS, Android | Modern teams, new deployments | Excellent |
| **OpenVPN** | 400-600 Mbps | Moderate (20 min) | All platforms (most mature) | Teams with legacy Windows | Good |
| **IKEv2/IPSec** | 700+ Mbps | Complex (45 min) | Windows, Mac, iOS, Android | Enterprise mixed environments | Excellent |
| **Tailscale (WireGuard-based)** | 800+ Mbps | Minimal (2 min) | All platforms | Remote teams, zero-trust networks | Excellent |

**For most remote teams:** WireGuard + modern Linux distribution = best balance of simplicity and security.

## Hardware Requirements

**Minimum Server Specs (10 concurrent users):**
- 2 vCPU, 2GB RAM (AWS t3.small or equivalent)
- 10 Mbps internet uplink (shared between all users)
- Static IP address for VPN endpoint

**Scaling Estimates:**
- 10-50 users: 1x t3.small (~$25-30/month)
- 50-200 users: 2x t3.medium + load balancer (~$80-120/month)
- 200+ users: 4x t3.large + geo-distributed load balancer (~$200-400/month)

## Step-by-Step: WireGuard VPN Setup

WireGuard is recommended for new deployments due to simplicity and performance.

### Step 1: Provision VPN Server (Ubuntu 22.04)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install WireGuard
sudo apt install wireguard wireguard-tools resolvconf -y

# Generate server key pair
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key

# View server public key (you'll share this with clients)
sudo cat /etc/wireguard/server_public.key
```

### Step 2: Create WireGuard Configuration

```bash
# Create configuration file
sudo nano /etc/wireguard/wg0.conf
```

**Configuration (replace 123.45.67.89 with your server's public IP):**

```ini
[Interface]
PrivateKey = <contents of /etc/wireguard/server_private.key>
ListenPort = 51820
Address = 10.0.0.1/24

# Enable IP forwarding for split-tunnel DNS
PostUp = iptables -A FORWARD -i %i -j ACCEPT
PostUp = iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -o %i -j ACCEPT

# DNS server routing (optional, see Step 5)
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Save configuration
UMask = 0077
SaveConfig = false
```

### Step 3: Add Client Configurations

For each remote user, generate a key pair and add to server config:

```bash
#!/bin/bash
# Script: add-wireguard-peer.sh

# Example: ./add-wireguard-peer.sh "alice@company.com"
CLIENT_NAME=$1
CLIENT_IP="10.0.0.$((2 + RANDOM % 250))"

# Generate client keys
CLIENT_PRIVATE=$(wg genkey)
CLIENT_PUBLIC=$(echo $CLIENT_PRIVATE | wg pubkey)

# Add to server config
sudo tee -a /etc/wireguard/wg0.conf > /dev/null <<EOF

# Peer: $CLIENT_NAME
[Peer]
PublicKey = $CLIENT_PUBLIC
AllowedIPs = $CLIENT_IP/32
EOF

# Create client config file
mkdir -p ~/wireguard-clients
cat > ~/wireguard-clients/${CLIENT_NAME}.conf <<EOF
[Interface]
PrivateKey = $CLIENT_PRIVATE
Address = $CLIENT_IP/32
DNS = 8.8.8.8

[Peer]
PublicKey = $(sudo cat /etc/wireguard/server_public.key)
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
Endpoint = 123.45.67.89:51820
PersistentKeepalive = 25
EOF

echo "Created config for $CLIENT_NAME at ~/wireguard-clients/${CLIENT_NAME}.conf"
```

**Usage:**
```bash
chmod +x add-wireguard-peer.sh
./add-wireguard-peer.sh "alice@company.com"
./add-wireguard-peer.sh "bob@company.com"

# Reload WireGuard after adding peers
sudo systemctl reload wg-quick@wg0
```

### Step 4: Enable WireGuard Service

```bash
# Enable IP forwarding (required for routing)
sudo sysctl -w net.ipv4.ip_forward=1
sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf

# Start WireGuard
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Verify it's running
sudo wg show
```

### Step 5: Configure Split-Tunnel DNS (Critical!)

Split-tunnel DNS ensures `internal.company.com` routes to your internal DNS, while public domains go to public DNS.

```bash
# On VPN server, install dnsmasq
sudo apt install dnsmasq -y

# Configure dnsmasq
sudo nano /etc/dnsmasq.conf
```

**Add these lines:**

```ini
# Listen on VPN interface
interface=wg0

# Route internal domains to your internal DNS server
server=/internal.company.com/192.168.1.1
server=/company.com/192.168.1.1

# Other domains go to public DNS
server=8.8.8.8
server=8.8.4.4

# Enable DNSSEC
dnssec
```

**Restart dnsmasq:**

```bash
sudo systemctl restart dnsmasq

# Verify dnsmasq is running on wg0
sudo netstat -tlnp | grep dnsmasq
```

**Update WireGuard client configs to use VPN gateway as DNS:**

```ini
# In each client config file
[Interface]
PrivateKey = <private-key>
Address = 10.0.0.X/32
DNS = 10.0.0.1  # Points to dnsmasq on VPN server
```

### Step 6: Firewall Configuration

Allow only internal resources through VPN, deny others:

```bash
# Allow WireGuard port
sudo ufw allow 51820/udp

# Allow internal traffic (10.0.0.0/24 = VPN subnet)
sudo ufw allow in on wg0

# Block VPN clients from accessing external services
# (only internal.company.com is accessible)
sudo iptables -t nat -A POSTROUTING -o wg0 -d 0.0.0.0/0 -j REJECT
```

**More restrictive: Only allow specific internal subnets**

```bash
# Allow VPN access to internal network only
sudo iptables -A FORWARD -i wg0 -d 192.168.1.0/24 -j ACCEPT
sudo iptables -A FORWARD -i wg0 -j DROP

# Save rules (persists after reboot)
sudo iptables-save | sudo tee /etc/iptables/rules.v4
sudo systemctl restart netfilter-persistent
```

## Client Setup (macOS Example)

### Download and Install WireGuard

```bash
# Option 1: Homebrew
brew install wireguard-tools

# Option 2: Download from App Store (WireGuard by Jason A. Donenfeld)
```

### Add VPN Configuration

```bash
# Copy client config from server
# Create ~/.config/wireguard/ directory
mkdir -p ~/.config/wireguard

# Add configuration file
nano ~/.config/wireguard/company-vpn.conf
# Paste contents from ~/wireguard-clients/alice@company.com.conf

# Import into WireGuard GUI
# Open WireGuard app → Click "+" → Select company-vpn.conf
```

### Connect to VPN

```bash
# Command line
sudo wg-quick up company-vpn

# Check connection status
sudo wg show

# Disconnect
sudo wg-quick down company-vpn
```

## Common VPN Issues and Fixes

### Issue 1: DNS Not Resolving Internal Domains

**Problem:** `ping internal.company.com` fails, but other VPN traffic works.

```bash
# Diagnosis
nslookup internal.company.com
# Output: Server: 10.0.0.1  (wrong — should be your DNS server)

# Fix: Verify DNS configuration on VPN server
sudo cat /etc/dnsmasq.conf | grep "server=/internal"

# Restart dnsmasq
sudo systemctl restart dnsmasq

# On client, verify DNS resolution
# macOS:
scutil --dns | grep "nameserver\[0\]"

# Linux:
sudo systemctl restart systemd-resolved
```

### Issue 2: Slow VPN Speed (Should be 400+ Mbps)

```bash
# Test speed through VPN
# Connect to VPN first, then run:
iperf3 -c <internal-server-ip> -R

# If speed is 50 Mbps or less:
# Check WireGuard MTU (max 1420 bytes)
ip link show wg0 | grep mtu

# Fix (if MTU too high)
ip link set dev wg0 mtu 1420

# Check server CPU (CPU-bound at high load)
ssh vpn-server "top -b -n 1 | head -20"
# If CPU > 80%, upgrade server instance size
```

### Issue 3: Intermittent Disconnects

```bash
# Problem: VPN drops every 30 minutes or 2 hours

# Fix 1: Enable PersistentKeepalive (in client config)
[Peer]
PersistentKeepalive = 25  # Sends keepalive every 25 seconds

# Fix 2: Check firewall rules (may timeout idle connections)
# Contact ISP/firewall provider if PersistentKeepalive doesn't help

# Fix 3: Increase server timeout
sudo nano /etc/sysctl.conf
# Add: net.netfilter.nf_conntrack_tcp_timeout_established = 3600
sudo sysctl -p

# Reconnect and test
sudo wg-quick down company-vpn && sleep 2 && sudo wg-quick up company-vpn
```

### Issue 4: Split-Tunnel DNS Breaks Public Websites

**Problem:** `google.com` resolves to internal IP or doesn't resolve.

```bash
# Diagnosis: Check what DNS is returning
nslookup google.com

# Fix: Verify dnsmasq configuration
# Should only route internal.company.com to internal DNS
sudo cat /etc/dnsmasq.conf | grep "server="

# If google.com is being routed internally, remove that line
sudo nano /etc/dnsmasq.conf
# Delete: server=192.168.1.1  (should route only company.com domains)
# Keep: server=8.8.8.8  (public DNS)

sudo systemctl restart dnsmasq
```

## Monitoring and Troubleshooting

**Check connected peers (on VPN server):**

```bash
sudo wg show
# Output shows which clients are connected, their IPs, and last handshake

# Example output:
# peer: ABc1dEfGhIJklMnOpQrStUvWxYzAbCdEfGhIjKl=
#   endpoint: 203.0.113.50:54322
#   allowed ips: 10.0.0.2/32
#   latest handshake: 5 minutes, 32 seconds ago
#   transfer: 1.42 MB received, 8.29 MB sent
#   persistent keepalive: 25 seconds
```

**Monitor bandwidth usage:**

```bash
# Real-time bandwidth per peer
watch -n 1 'sudo wg show'

# Or use vnstat (if installed)
sudo apt install vnstat
vnstat -i wg0
```

**View VPN logs:**

```bash
# Check system logs for errors
sudo journalctl -u wg-quick@wg0 -n 50

# Watch real-time logs
sudo journalctl -u wg-quick@wg0 -f
```

## Alternative: Tailscale (Faster Setup)

If you prefer managed VPN over self-hosted, Tailscale uses WireGuard under the hood:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate
sudo tailscale up --advertise-routes=192.168.1.0/24

# No manual configuration needed — Tailscale handles routing, DNS, firewall

# Cost: Free for <100 devices, then $3-10/month per additional device
```

**Pros:** Easy setup, automatic DNS, works across NATs
**Cons:** Data routed through Tailscale's infrastructure (not private by default)

## Cost Comparison

| Setup | Hardware | Annual Cost | Setup Time | Maintenance |
|-------|----------|------------|-----------|-------------|
| **Self-hosted WireGuard** | VPS (t3.small) | $300-400 | 1-2 hours | 1-2 hrs/month |
| **Self-hosted OpenVPN** | VPS (t3.small) | $300-400 | 2-3 hours | 2-3 hrs/month |
| **Tailscale Free** | None | $0 | 5 minutes | Minimal |
| **Tailscale Pro** | None | $120-300/year | 5 minutes | Minimal |
| **Corporate VPN (Cisco, Fortinet)** | Hardware | $5,000-20,000+ | 40+ hours | Full-time support |

For teams <100: Self-hosted WireGuard or Tailscale
For teams >100: Self-hosted with load balancing or enterprise solution

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Related Reading

Explore more remote work infrastructure at [Remote Work Tools Guides Hub](/remote-work-tools/guides-hub/)
