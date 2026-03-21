---
layout: default
title: "Linux Server Hardening Guide for Remote Developers"
description: "Harden a Linux VPS or home lab server for remote development use. Covers SSH key auth, UFW firewall, fail2ban, unattended upgrades, and audit logging setup."
date: 2026-03-21
author: theluckystrike
permalink: /linux-server-hardening-remote-developers/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A freshly provisioned VPS or home lab server is exposed to the internet with default settings — password authentication enabled, all ports open, no rate limiting. Within minutes of provisioning, automated scanners are probing it. This guide hardens a Ubuntu 22.04/24.04 server to a baseline that remote developers can use with confidence.

All commands run as root or with sudo unless otherwise noted.

## Step 1: Create a Non-Root User

Never use the `root` user for routine work. Create a deploy user and disable root login.

```bash
# Create admin user
adduser devadmin
usermod -aG sudo devadmin

# Set up SSH keys for devadmin
mkdir -p /home/devadmin/.ssh
chmod 700 /home/devadmin/.ssh

# Copy your public key
# On your local machine:
cat ~/.ssh/id_ed25519.pub

# On the server, add to authorized_keys
echo "ssh-ed25519 AAAAC3Nz... your-key-comment" \
  >> /home/devadmin/.ssh/authorized_keys
chmod 600 /home/devadmin/.ssh/authorized_keys
chown -R devadmin:devadmin /home/devadmin/.ssh
```

Test SSH login with the new user before disabling root login:

```bash
# From your local machine — test before locking root out
ssh devadmin@your-server-ip
sudo whoami  # should return "root"
```

## Step 2: Harden SSH Configuration

```bash
# /etc/ssh/sshd_config — edit these settings
sudo nano /etc/ssh/sshd_config
```

```ini
# /etc/ssh/sshd_config

# Disable root login completely
PermitRootLogin no

# Disable password authentication — keys only
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no

# Disable forwarding if you don't need it
X11Forwarding no
AllowTcpForwarding yes    # needed for SSH tunnels to databases etc.

# Change default port (optional — reduces log noise, not a security measure)
Port 2222

# Only allow specific users
AllowUsers devadmin deploy

# Idle session timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# Use modern key exchange algorithms only
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
```

```bash
# Test config before restarting
sudo sshd -t

# Restart SSH
sudo systemctl restart ssh
```

If you changed the port, update your local `~/.ssh/config`:

```bash
# ~/.ssh/config on your local machine
Host myserver
  HostName your-server-ip
  User devadmin
  Port 2222
  IdentityFile ~/.ssh/id_ed25519
```

## Step 3: Configure UFW Firewall

```bash
# Install and configure UFW (Uncomplicated Firewall)
sudo apt install ufw -y

# Set defaults: deny all incoming, allow all outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (use the port you set in sshd_config)
sudo ufw allow 2222/tcp comment "SSH"

# Allow specific services as needed
sudo ufw allow 80/tcp comment "HTTP"
sudo ufw allow 443/tcp comment "HTTPS"

# Allow access only from a specific IP (e.g., monitoring server)
sudo ufw allow from 192.168.1.100 to any port 9100 comment "Prometheus node exporter"

# Allow WireGuard
sudo ufw allow 51820/udp comment "WireGuard"

# Enable UFW
sudo ufw enable
sudo ufw status verbose
```

## Step 4: Install and Configure fail2ban

fail2ban reads log files and bans IPs that show malicious patterns (e.g., repeated failed SSH logins).

```bash
sudo apt install fail2ban -y

# Create local config (don't edit jail.conf — it gets overwritten on updates)
sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port    = 2222
logpath = %(sshd_log)s
maxretry = 3
bantime = 24h

[sshd-ddos]
enabled  = true
port     = 2222
logpath  = %(sshd_log)s
maxretry = 10
findtime = 2m
bantime  = 24h
EOF

sudo systemctl enable fail2ban
sudo systemctl restart fail2ban

# Check status
sudo fail2ban-client status sshd

# Unban an IP if you accidentally banned yourself
sudo fail2ban-client set sshd unbanip YOUR_IP
```

## Step 5: Automatic Security Updates

```bash
sudo apt install unattended-upgrades -y

# /etc/apt/apt.conf.d/50unattended-upgrades
sudo tee /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

// Auto-remove unused packages
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// Reboot if required (at 2am)
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";

// Email on errors
Unattended-Upgrade::Mail "you@yourdomain.com";
Unattended-Upgrade::MailReport "on-change";
EOF

# Enable automatic updates
sudo tee /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
EOF

# Test the configuration
sudo unattended-upgrade --dry-run --debug
```

## Step 6: Configure auditd for Logging

auditd logs privileged commands, file access, and user logins — useful for forensics if something goes wrong.

```bash
sudo apt install auditd -y

# /etc/audit/rules.d/hardening.rules
sudo tee /etc/audit/rules.d/hardening.rules << 'EOF'
# Delete all existing rules
-D

# Monitor sudoers changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes

# Monitor SSH authorized_keys changes
-w /home -p wa -k home_changes
-w /root/.ssh -p wa -k ssh_changes

# Monitor network configuration changes
-w /etc/hosts -p wa -k network_changes
-w /etc/network/ -p wa -k network_changes

# Log all sudo commands
-a always,exit -F arch=b64 -S execve -F uid=root -F auid>=1000 -F auid!=-1 -k root_commands

# Monitor cron changes
-w /etc/cron.d/ -p wa -k cron_changes
-w /etc/crontab -p wa -k cron_changes
EOF

sudo auditctl -R /etc/audit/rules.d/hardening.rules
sudo systemctl enable auditd
sudo systemctl restart auditd

# Query audit logs
sudo ausearch -k sudoers_changes -i  # recent sudoers changes
sudo ausearch -k root_commands -i    # commands run as root
sudo aureport --logins --summary     # login summary report
```

## Step 7: Kernel Security Settings

```bash
# /etc/sysctl.d/99-hardening.conf
sudo tee /etc/sysctl.d/99-hardening.conf << 'EOF'
# Disable IP forwarding (enable only if this is a router/VPN server)
# net.ipv4.ip_forward = 0

# Prevent SYN flood attacks
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# Prevent ICMP redirect attacks
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0

# Log martian packets (invalid source addresses)
net.ipv4.conf.all.log_martians = 1

# Restrict dmesg to root
kernel.dmesg_restrict = 1

# Disable core dumps with setuid programs
fs.suid_dumpable = 0
EOF

sudo sysctl --system
```

## Verification Checklist

```bash
# Run after hardening to verify settings
echo "=== SSH Config ==="
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|port"

echo "=== UFW Status ==="
sudo ufw status numbered

echo "=== fail2ban Status ==="
sudo fail2ban-client status sshd

echo "=== Unattended Upgrades ==="
sudo unattended-upgrade --dry-run 2>&1 | tail -5

echo "=== auditd Status ==="
sudo systemctl is-active auditd

echo "=== Listening Ports ==="
sudo ss -tlnp

echo "=== Open world ports (should only show intended services) ==="
sudo nmap -sV --open -p- localhost 2>/dev/null | grep "open"
```

## Related Reading

- [Home Lab Setup Guide for Remote Developers](/remote-work-tools/home-lab-setup-guide-remote-developers/)
- [WireGuard Team VPN: Multi-User Setup Guide](/remote-work-tools/wireguard-team-vpn-multi-user-setup/)
- [Prometheus Monitoring Setup for Remote Infrastructure](/remote-work-tools/prometheus-monitoring-remote-infrastructure/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
