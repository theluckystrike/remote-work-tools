---
layout: default
title: "Remote Work VoIP Setup for Home Offices"
description: "Configure a reliable VoIP system for remote home offices using FreePBX, Linphone, and quality-of-service settings for clear business calls"
date: 2026-03-22
author: theluckystrike
permalink: /remote-work-voip-setup-for-home-offices/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---
{% raw %}

A proper VoIP setup replaces desk phones with software-based calling that works from any home office. This guide covers a self-hosted FreePBX deployment, softphone configuration, QoS tuning, and failover so remote workers maintain business call quality.

## Architecture Overview

```
Internet
    │
    ▼
SIP Trunk Provider (Twilio/VoIP.ms/Bandwidth)
    │
    ▼
FreePBX Server (cloud VM or on-prem)
    │
    ├── Extension 101 → Linphone (desktop)
    ├── Extension 102 → Zoiper (mobile)
    └── Extension 103 → Hardware IP phone
```

## FreePBX Installation

Deploy on Ubuntu 22.04 (2 vCPU, 2GB RAM minimum):

```bash
# Download FreePBX installer
wget https://github.com/FreePBX/sng_freepbx_debian_install/raw/master/sng_freepbx_debian_install.sh

# Run installer (takes 20-30 minutes)
chmod +x sng_freepbx_debian_install.sh
sudo bash sng_freepbx_debian_install.sh

# Verify Asterisk is running
sudo systemctl status asterisk
fwconsole sa
```

## Firewall Rules

```bash
# UFW rules for SIP and RTP
sudo ufw allow 5060/udp   # SIP signaling
sudo ufw allow 5061/tcp   # SIP TLS
sudo ufw allow 10000:20000/udp  # RTP audio ports
sudo ufw allow 80/tcp     # HTTP admin
sudo ufw allow 443/tcp    # HTTPS admin

# Restrict admin access to your team's IPs
sudo ufw allow from 203.0.113.0/24 to any port 80
sudo ufw allow from 203.0.113.0/24 to any port 443
sudo ufw deny 80/tcp
sudo ufw deny 443/tcp
```

## SIP Trunk Configuration

In FreePBX admin (Connectivity > Trunks):

```ini
# Trunk settings for VoIP.ms (example)
[voipms-trunk]
type=friend
host=atlanta1.voip.ms
port=5060
username=your_account_number
secret=your_password
insecure=invite,port
qualify=yes
context=from-trunk
disallow=all
allow=ulaw,alaw,g722
dtmfmode=rfc2833
nat=force_rport,comedia
```

For Twilio SIP Trunking:

```bash
# Configure via Twilio Console, then add trunk in FreePBX:
# Trunk Type: SIP
# Outbound CallerID: +15551234567
# SIP Server: your-subdomain.pstn.twilio.com
# Username: your_twilio_account_sid
# Secret: your_twilio_auth_token
```

## Extension Configuration

```ini
# /etc/asterisk/sip_custom.conf
[101]
type=friend
host=dynamic
secret=str0ng-ext-password
username=101
callerid=Alice Smith <101>
context=from-internal
dtmfmode=rfc2833
nat=force_rport,comedia
qualify=yes
disallow=all
allow=ulaw,g722
transport=tls,udp
encryption=yes

[102]
type=friend
host=dynamic
secret=str0ng-ext-password-2
username=102
callerid=Bob Jones <102>
context=from-internal
dtmfmode=rfc2833
nat=force_rport,comedia
qualify=yes
disallow=all
allow=ulaw,g722
```

## Linphone Softphone Setup

Install on Linux/Mac/Windows:

```bash
# Ubuntu
sudo apt install linphone

# macOS
brew install --cask linphone

# Configure via CLI or settings file
cat ~/.linphonerc
```

Settings to configure in Linphone:

```
Preferences > SIP Accounts > Add Account

SIP Address: "sip:101@your-pbx.example.com"
SIP Password:      str0ng-ext-password
SIP Server: "your-pbx.example.com:5060"
Transport:         TLS (recommended)
STUN server: "stun.l.google.com:19302"
Enable ICE:        Yes
SRTP:              Mandatory
```

## Zoiper Mobile Configuration

```
Account Type: SIP
Username: 102
Password: str0ng-ext-password-2
Domain: your-pbx.example.com
Port: 5061
Transport: TLS
SRTP: Required
STUN: stun.l.google.com
```

## Router QoS Configuration

QoS prevents audio dropouts when bandwidth is shared. Access your router admin panel:

```bash
# On pfSense/OPNsense - ALTQ/HFSC QoS
# Traffic shaper > HFSC

# Create a VoIP queue with:
# - Priority: Highest
# - Bandwidth guarantee: 128kbps per call (g.711)
# - Match rule: UDP port 10000-20000 (RTP)
# - Match rule: UDP port 5060 (SIP)

# On Linux router/gateway using tc
# Mark SIP/RTP packets
iptables -t mangle -A PREROUTING -p udp --dport 5060 -j DSCP --set-dscp-class EF
iptables -t mangle -A PREROUTING -p udp --dport 10000:20000 -j DSCP --set-dscp-class EF

# Apply QoS with HTB
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: "classid 1:1 htb rate 100mbit"
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 5mbit ceil 10mbit prio 1  # VoIP
tc class add dev eth0 parent 1:1 classid 1:30 htb rate 90mbit ceil 100mbit prio 3 # Default
tc filter add dev eth0 parent 1: "protocol ip handle 0x2e fw classid 1:10"
```

## Fail2ban for SIP Security

```bash
# /etc/fail2ban/filter.d/asterisk.conf
[Definition]
failregex = NOTICE.* .*: Registration from '.*' failed for '<HOST>.*' - Wrong password
            NOTICE.* .*: Registration from '.*' failed for '<HOST>.*' - No matching peer
            NOTICE.* <HOST> failed to authenticate as '.*'$

ignoreregex =
```

```ini
# /etc/fail2ban/jail.local
[asterisk]
enabled  = true
port     = 5060,5061
filter   = asterisk
logpath  = /var/log/asterisk/full
maxretry = 5
bantime  = 3600
findtime = 300
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status asterisk
```

## Call Quality Testing

```bash
# Test codec performance
asterisk -rx "sip show peers"

# Check active calls
asterisk -rx "core show channels"

# Monitor RTP quality
asterisk -rx "rtp set debug on"
asterisk -rx "rtp set debug off"

# Network latency check for VoIP (must be <150ms for good quality)
ping -i 0.2 -c 50 your-pbx.example.com | tail -1
# Target: avg < 80ms, max < 150ms
```

## Hunt Groups and IVR

```bash
# In FreePBX: Applications > Ring Groups
# Ring Group Number: 600
# Ring Strategy: ringall
# Ring Time: 20 seconds
# Extensions: 101-102-103
# Destination if no answer: Voicemail

# IVR via Applications > IVR
# Press 1 -> Extension 101 (Sales)
# Press 2 -> Extension 102 (Support)
# Press 0 -> Ring Group 600 (General)
```

## Monitoring and Uptime

```bash
# Check Asterisk status
fwconsole sa

# Restart if needed
sudo systemctl restart asterisk

# Cron-based health check
# Add to crontab -e
*/5 * * * * asterisk -rx "core show version" > /dev/null 2>&1 || systemctl restart asterisk

# SIP trunk registration status
asterisk -rx "sip show registry"
# Output: Host: atlanta1.voip.ms    State: Registered
```

## Related Reading

- [Best Headset for Remote Work Video Calls](/remote-work-tools/best-headset-for-remote-work-video-calls/)
- [Best Noise Cancelling Microphones for Home Offices](/remote-work-tools/best-noise-cancelling-microphones-for-home-offices-busy-streets/)
- [Best Remote Work Network Diagnostic Toolkit](/remote-work-tools/remote-work-network-diagnostic-toolkit/)
- [Best Desk for Corner Home Office Room Layout Setup 2026](/remote-work-tools/best-desk-for-corner-home-office-room-layout-setup-2026/)
---

## Related Articles

- [Home Lab Setup Guide for Remote Developers](/remote-work-tools/home-lab-setup-guide-remote-developers/)
- [How to Set Up Home Office Network for Remote Work](/remote-work-tools/how-to-set-up-home-office-network-for-remote-work/)
- [VS Code Remote Development Setup Guide](/remote-work-tools/vscode-remote-development-setup/)
- [How to Set Up HIPAA Compliant Home Office for Remote](/remote-work-tools/how-to-set-up-hipaa-compliant-home-office-for-remote-healthc/)
- [Remote Developer Home Office Monitor Setup Guide](/remote-work-tools/remote-developer-home-office-monitor-setup-guide-ultrawide-vs-dual/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
