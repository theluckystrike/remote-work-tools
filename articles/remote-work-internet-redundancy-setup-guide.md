---
layout: default
title: "Remote Work Internet Redundancy Setup Guide"
description: "Build a failover internet setup for remote engineers using 4G/5G backup, router failover configuration, and tools that minimize connection disruption"
date: 2026-03-22
author: theluckystrike
permalink: /remote-work-internet-redundancy-setup-guide/
categories: [guides]
tags: [remote-work-tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

A single ISP connection is a single point of failure. For engineers on customer calls, async video reviews, or live deployments, a dropped connection at the wrong moment costs trust and time. This guide covers a practical dual-ISP failover setup for home offices that achieves automatic failover in under 30 seconds.

## The Core Setup

The goal is two independent internet connections that switch automatically when the primary fails:

```
Primary ISP (fiber/cable)
        ↓
    Router with
   dual-WAN failover  ──→  Your devices
        ↓
Secondary ISP (4G/5G cellular)
```

Hardware that supports this natively: Firewalla Gold Plus, GL.iNet Flint 2, Peplink Balance One (prosumer), or a Mikrotik RouterOS setup.

## Hardware Option 1: GL.iNet Flint 2 (Budget)

The GL-MT6000 runs OpenWrt and supports WAN failover out of the box for ~$100.

```bash
# SSH into the router after initial setup
ssh root@192.168.8.1

# Check current WAN status
uci show network.wan
uci show network.wan6

# Configure the secondary WAN (USB tethering from phone or USB modem)
# Flint 2 supports USB tethering natively through the UI
# Go to: Network → Internet → Add → USB Tethering
```

**Failover configuration via OpenWrt UCI:**

```bash
# Set up mwan3 (multi-WAN manager)
opkg update && opkg install mwan3 luci-app-mwan3

# Configure tracking targets for each WAN
cat > /etc/config/mwan3 << 'EOF'
config globals 'globals'
    option mmx_mask '0x3F00'

config interface 'wan'
    option enabled 1
    option track_ip '8.8.8.8 1.1.1.1'
    option reliability 1
    option count 2
    option timeout 2
    option interval 5
    option down 3
    option up 8

config interface 'wanb'
    option enabled 1
    option track_ip '8.8.8.8 1.1.1.1'
    option reliability 1
    option count 2
    option timeout 2
    option interval 5
    option down 3
    option up 8

config rule 'default_rule'
    option sticky 1
    option use_policy 'failover'

config policy 'failover'
    option use_member 'wan_100'

config policy 'failover'
    list use_member 'wan_100'
    list use_member 'wanb_200'

config member 'wan_100'
    option interface wan
    option metric 1
    option weight 100

config member 'wanb_200'
    option interface wanb
    option metric 2
    option weight 100
EOF

service mwan3 restart
```

## Hardware Option 2: Peplink Balance One ($299)

Purpose-built for dual-WAN failover with a simpler UI. Plug in both connections, enable SpeedFusion health checks, done.

**Key settings:**
- Health Check: HTTP/HTTPS to `www.gstatic.com` every 5 seconds
- Failover to secondary when primary misses 3 consecutive checks
- Recovery: switch back to primary after 8 consecutive successes

The 3-miss / 8-success asymmetry prevents flapping on an unstable primary connection.

## 4G/5G Backup Modem Recommendations

| Device | Band Coverage | Speed | Monthly |
|---|---|---|---|
| Netgear M6 Pro | 5G mmWave + Sub-6 | 4Gbps theoretical | SIM-based |
| GL.iNet Mudi v2 | 4G LTE | 150Mbps | SIM-based |
| Solis Lite | 4G LTE, global | 50Mbps | 3GB free/day |
| Phone USB tethering | 4G/5G (carrier) | Depends | Existing plan |

For most remote engineers, phone USB tethering is the cheapest backup — most carrier plans include tethering at no extra cost. The latency is higher than fiber but sufficient for SSH, async video, and Slack.

## Testing Failover Behavior

```bash
# Install mtr for continuous path monitoring
brew install mtr  # macOS
# or: apt install mtr-tiny  # Linux

# Watch the path to Google DNS in real time
sudo mtr 8.8.8.8 --report-cycles 1000 --interval 0.5

# Simulate primary failure: unplug the cable or disable WAN1 in router UI
# Watch mtr — you should see packet loss for 15-30 seconds, then recovery via WAN2
```

Monitor the transition time. Acceptable: under 30 seconds. Unacceptable: over 90 seconds (indicates health check intervals are too long or recovery threshold is too high).

## Application-Level Failover Gaps

Automatic failover at the router level doesn't fix everything. Stateful connections break:

- **SSH sessions**: use `mosh` instead of `ssh` — it reconnects automatically
- **Video calls**: Zoom/Meet reconnect themselves within 30 seconds
- **VPN tunnels**: most split automatically, but check your VPN client settings
- **Database connections**: poolers like PgBouncer reconnect; raw `psql` sessions drop

```bash
# Install mosh for SSH resilience
brew install mosh  # macOS client
apt install mosh   # server

# Connect with mosh instead of ssh
mosh user@server.example.com

# mosh keeps your session alive through IP changes and reconnects transparently
```

## Monitoring Connection Health

```bash
# Simple cron-based uptime logger
cat > /usr/local/bin/check-internet.sh << 'EOF'
#!/bin/bash
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
if curl -s --max-time 5 https://www.google.com > /dev/null; then
    echo "$TIMESTAMP OK"
else
    echo "$TIMESTAMP DOWN"
fi
EOF
chmod +x /usr/local/bin/check-internet.sh

# Add to crontab: check every minute
echo "* * * * * /usr/local/bin/check-internet.sh >> /var/log/internet-uptime.log" | crontab -

# Calculate monthly uptime
awk '/DOWN/ {down++} /OK/ {up++} END {print "Uptime: " up/(up+down)*100 "%"}' /var/log/internet-uptime.log
```

## Budget Breakdown

| Component | Cost |
|---|---|
| GL.iNet Flint 2 router | $100 |
| Netgear M6 Pro 5G modem | $249 |
| Prepaid SIM (3GB/day) | $25-40/month |
| Total monthly | $25-40 + existing ISP |

For a $40/month total add-on, you eliminate the most common cause of remote work disruption.

## Configuring DNS Resilience Alongside Internet Redundancy

Dual-WAN failover handles the physical link layer, but DNS failures can make the internet appear down even when your connection is working. Remote workers operating from home offices with a single DNS resolver (typically provided by the ISP) experience DNS outages during partial connectivity issues even when raw packet routing is functional.

Configure your router or devices to use multiple DNS resolvers with fallback logic. On a GL.iNet router with OpenWrt, override the default ISP DNS:

```bash
# SSH into router
ssh root@192.168.8.1

# Configure DNS with multiple upstream resolvers
uci set dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='8.8.8.8'
uci add_list dhcp.@dnsmasq[0].server='9.9.9.9'
uci set dhcp.@dnsmasq[0].noresolv='1'
uci commit dhcp
service dnsmasq restart

# Verify DNS is resolving through both servers
nslookup google.com 1.1.1.1
nslookup google.com 8.8.8.8
```

For devices where you cannot control the router, configure DNS-over-HTTPS directly on macOS or Linux to bypass ISP DNS entirely:

```bash
# Install cloudflared for DNS-over-HTTPS on macOS
brew install cloudflare/cloudflare/cloudflared

# Create LaunchDaemon for automatic startup
sudo cloudflared service install --legacy

# Configure to proxy DNS requests locally
sudo tee /Library/LaunchDaemons/com.cloudflare.cloudflared.plist > /dev/null << 'EOF'
<key>ProgramArguments</key>
<array>
    <string>/usr/local/bin/cloudflared</string>
    <string>proxy-dns</string>
    <string>--port</string>
    <string>5053</string>
    <string>--upstream</string>
    <string>https://1.1.1.1/dns-query</string>
    <string>--upstream</string>
    <string>https://1.0.0.1/dns-query</string>
</array>
EOF

# Point system DNS to local resolver
networksetup -setdnsservers "Wi-Fi" 127.0.0.1
```

This setup means DNS queries succeed as long as either your primary or secondary internet connection works, and they are encrypted against ISP inspection regardless of which connection is active.

## Automating Failover Notifications

Knowing when failover occurred is valuable for diagnosing patterns—if you fail over to cellular every day between 9 and 10 AM, that signals a recurring ISP issue worth reporting. Build a simple notification system that alerts you when the active WAN changes:

```bash
# /usr/local/bin/wan-monitor.sh
# Run via cron every 2 minutes on the router or a local machine

#!/bin/bash
STATE_FILE="/tmp/wan-state"
CURRENT_WAN=$(curl -s --max-time 3 --interface eth0 https://ipinfo.io/ip 2>/dev/null)
BACKUP_WAN=$(curl -s --max-time 3 --interface wwan0 https://ipinfo.io/ip 2>/dev/null)

# Check which WAN is currently active by testing connectivity
if curl -s --max-time 5 --interface eth0 https://www.google.com > /dev/null 2>&1; then
    ACTIVE="primary"
else
    ACTIVE="backup"
fi

PREV=$(cat "$STATE_FILE" 2>/dev/null || echo "primary")

if [ "$ACTIVE" != "$PREV" ]; then
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    echo "$TIMESTAMP: WAN switched from $PREV to $ACTIVE" >> /var/log/wan-failover.log

    # Send notification via ntfy.sh (free push notification service)
    curl -s -d "WAN switched to $ACTIVE at $TIMESTAMP" \
        https://ntfy.sh/your-unique-channel-name > /dev/null

    echo "$ACTIVE" > "$STATE_FILE"
fi
```

```bash
# Add to crontab
echo "*/2 * * * * /usr/local/bin/wan-monitor.sh" | crontab -

# Subscribe to notifications on your phone via the ntfy app
# Channel: your-unique-channel-name (use a hard-to-guess string)
```

This gives you a historical log of failover events and real-time mobile notifications. After a week, review `/var/log/wan-failover.log` to identify patterns in your ISP's reliability.

## WireGuard VPN Across Dual-WAN

Remote engineers often use VPNs for accessing office infrastructure. Standard VPN connections bind to a specific IP address and drop when that address changes during failover. WireGuard handles this better than OpenVPN or IPSec because it uses UDP and re-establishes connections quickly after an IP change.

Configure WireGuard to reconnect automatically after failover:

```bash
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 10.0.0.1

# Keep WireGuard alive through IP changes
PostUp = wg set wg0 peer PEER_PUBLIC_KEY endpoint your-vpn-server.com:51820
PostDown = echo "WireGuard down"

[Peer]
PublicKey = PEER_PUBLIC_KEY
Endpoint = your-vpn-server.com:51820
AllowedIPs = 10.0.0.0/24, 172.16.0.0/12
PersistentKeepalive = 25
```

The `PersistentKeepalive = 25` setting sends a keepalive packet every 25 seconds, which maintains NAT table entries through connection switches. After failover, WireGuard reconnects within one keepalive interval—typically under 30 seconds—without requiring any user action.

## Related Reading

- [Best Backup Internet Solution for Remote Workers in Countries with Poor Fiber](/best-backup-internet-solution-for-remote-workers-in-countrie/)
- [Best Portable WiFi Hotspot for Digital Nomads](/best-portable-wifi-hotspot-for-digital-nomads/)
- [Best Ethernet Over Powerline Adapter for Home Office Far from Router](/best-ethernet-over-powerline-adapter-for-home-office-far-fro/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
