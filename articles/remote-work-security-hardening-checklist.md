---
layout: default
title: "Remote Work Security Hardening Checklist"
description: "Security hardening checklist for remote workers: SSH key setup, MFA, disk encryption, DNS-over-HTTPS, secrets management, and network security for developers."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /remote-work-security-hardening-checklist/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, security, remote-work]
---
---
layout: default
title: "Remote Work Security Hardening Checklist"
description: "Security hardening checklist for remote workers: SSH key setup, MFA, disk encryption, DNS-over-HTTPS, secrets management, and network security for developers."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /remote-work-security-hardening-checklist/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, security, remote-work]
---

{% raw %}

Remote work expands your attack surface. Your home network, personal laptop, and public WiFi hotspots are all less controlled than a corporate office environment. This checklist covers the practical security hardening steps every remote developer should have in place, with commands to verify and implement each one.

## SSH Key Security

Weak SSH keys are still how most servers get compromised. Audit and upgrade your keys:

```bash
# Check existing SSH keys
ls -la ~/.ssh/
ssh-keygen -l -f ~/.ssh/id_rsa    # if you have an RSA key, check the bit size
# 2048-bit RSA is acceptable; 4096 is better; switch to Ed25519

# Generate a strong Ed25519 key (recommended)
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519

# Generate with a passphrase (always use one for keys with server access)
# The above command prompts for passphrase

# Copy to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server.example.com

# Verify server only allows key auth (no passwords)
ssh user@server.example.com "sudo grep PasswordAuthentication /etc/ssh/sshd_config"
# Should show: PasswordAuthentication no

# Harden server sshd_config
sudo tee /etc/ssh/sshd_config.d/hardening.conf > /dev/null << 'EOF'
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 20
X11Forwarding no
AllowTcpForwarding yes
ClientAliveInterval 300
ClientAliveCountMax 2
EOF
sudo systemctl reload sshd
```

**Check your SSH agent is loaded with keys:**

```bash
# Start ssh-agent and add key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Add to ~/.bashrc/.zshrc for auto-load:
# if [ -z "$SSH_AUTH_SOCK" ]; then
#   eval "$(ssh-agent -s)"
#   ssh-add ~/.ssh/id_ed25519 2>/dev/null
# fi
```

## Multi-Factor Authentication (MFA)

Enable MFA everywhere. Priority order:

1. GitHub / GitLab — use TOTP or hardware key
2. AWS/GCP/Azure console
3. Your company SSO
4. Password manager (if not hardware-key protected)

```bash
# Set up TOTP MFA for SSH logins on Linux servers
sudo apt-get install libpam-google-authenticator

# Run setup as the user you log in as
google-authenticator
# Answer yes to all prompts
# Scan the QR code with Google Authenticator or Authy

# Configure PAM to require TOTP
sudo tee -a /etc/pam.d/sshd > /dev/null << 'EOF'
auth required pam_google_authenticator.so nullok
EOF

# Update sshd_config
sudo sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/' /etc/ssh/sshd_config
echo "AuthenticationMethods publickey,keyboard-interactive" | sudo tee -a /etc/ssh/sshd_config

sudo systemctl reload sshd
```

## Disk Encryption

Disk encryption protects data if your laptop is lost or stolen.

**macOS — FileVault:**

```bash
# Check FileVault status
sudo fdesetup status
# Should show: FileVault is On.

# Enable if off
sudo fdesetup enable

# Check encryption progress
sudo fdesetup status
```

**Linux — LUKS (setup at install time; check if active):**

```bash
# Check if disk is encrypted
lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT | grep crypt
# or
sudo cryptsetup status /dev/mapper/sda*

# Check for existing LUKS containers
sudo blkid | grep crypto_LUKS

# If not encrypted, you need to reinstall with encryption enabled
# Most modern Linux installers (Ubuntu, Fedora) offer this during setup
```

## Firewall Configuration

```bash
# macOS — check firewall status
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
# Enable: sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# Linux (UFW)
# Check status
sudo ufw status verbose

# Enable and set defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH only from your known IPs
sudo ufw allow from 203.0.113.0/24 to any port 22

# Allow specific services
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable
sudo ufw enable

# Block common attack vectors
sudo ufw deny 23/tcp    # telnet
sudo ufw deny 21/tcp    # FTP
sudo ufw deny 25/tcp    # SMTP (unless you run mail)
```

## DNS-over-HTTPS (DoH)

Plain DNS leaks which domains you visit. Use encrypted DNS, especially on untrusted networks:

```bash
# macOS: use Cloudflare's 1.1.1.1 with DoH
# System Preferences → Network → Advanced → DNS
# Add: 1.1.1.1 and 1.0.0.1
# Enable DoH in macOS 14+: System Settings → Privacy & Security → Local Network Privacy

# Linux: configure systemd-resolved with DoH
sudo tee /etc/systemd/resolved.conf.d/doh.conf > /dev/null << 'EOF'
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com
DNSOverTLS=yes
DNSSEC=yes
EOF

sudo systemctl restart systemd-resolved

# Verify DNS is working
resolvectl status | grep "DNS Server"
resolvectl query github.com

# Alternative: dnscrypt-proxy (more control)
sudo apt-get install dnscrypt-proxy
# Configure /etc/dnscrypt-proxy/dnscrypt-proxy.toml
# Set server_names = ['cloudflare', 'cloudflare-ipv6']
```

## Secrets Management

Never store secrets in dotfiles, shell history, or environment variables in plain text:

```bash
# Check shell history for accidental secret exposure
grep -E "(password|token|secret|key|api)" ~/.bash_history ~/.zsh_history 2>/dev/null | head -20
# If anything appears: clear specific entries or the full history

# Clear Bash history of secrets
HISTFILE=/dev/null  # disable history for current session
history -c           # clear in-memory history

# Better: use a secrets manager
# 1Password CLI
eval $(op signin)
export GITHUB_TOKEN=$(op read "op://Work/GitHub PAT/token")
export AWS_SECRET_ACCESS_KEY=$(op read "op://Work/AWS Dev/secret_access_key")

# Or use pass (GPG-encrypted password store)
sudo apt-get install pass gnupg2
gpg --gen-key
pass init YOUR_GPG_KEY_ID
pass insert github/token
export GITHUB_TOKEN=$(pass github/token)

# Add to ~/.bashrc for auto-loading secrets
# eval $(op signin --account myaccount) 2>/dev/null
# export GITHUB_TOKEN=$(op read "op://Work/GitHub PAT/token" 2>/dev/null)
```

**Block accidental git commits with secrets:**

```bash
# Install git-secrets
brew install git-secrets   # macOS
# or
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets && sudo make install

# Set up for AWS secrets detection in all repos
git secrets --register-aws --global

# Add custom patterns
git secrets --add --global 'ghp_[A-Za-z0-9]{36}'     # GitHub PAT
git secrets --add --global 'sk-[A-Za-z0-9]{48}'       # OpenAI key
git secrets --add --global 'xoxb-[A-Za-z0-9-]+'       # Slack bot token

# Apply hooks to existing repo
cd /path/to/repo
git secrets --install
```

## Public WiFi Precautions

```bash
# Check if you have a VPN running before connecting to public WiFi
# Quick check: is your traffic going through a VPN?
curl https://ipinfo.io | python3 -m json.tool
# Compare IP to your actual location — if it's different, VPN is active

# Use Tailscale as your VPN for public WiFi
sudo tailscale up --exit-node=your-exit-node-hostname

# Verify traffic is now through Tailscale exit node
curl https://ipinfo.io

# Disable when back on trusted network
sudo tailscale up --exit-node=
```

## Security Audit Script

Run this periodically to check your security posture:

```bash
#!/bin/bash
# security-audit.sh — quick remote work security check

echo "=== Remote Work Security Audit ==="
echo ""

# SSH Keys
echo "SSH Keys:"
for key in ~/.ssh/id_*; do
  if [[ "$key" != *.pub ]]; then
    echo "  $key: $(ssh-keygen -l -f "$key" 2>/dev/null | awk '{print $1, $4}')"
  fi
done

# Disk encryption (macOS)
if [[ "$OSTYPE" == "darwin"* ]]; then
  echo ""
  echo "Disk Encryption:"
  sudo fdesetup status 2>/dev/null | head -1
fi

# Firewall
echo ""
echo "Firewall:"
if [[ "$OSTYPE" == "darwin"* ]]; then
  sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate 2>/dev/null
else
  sudo ufw status 2>/dev/null | head -1
fi

# Open ports
echo ""
echo "Listening ports:"
ss -tlnp 2>/dev/null || netstat -tlnp 2>/dev/null | grep LISTEN

# Check for git-secrets
echo ""
echo "git-secrets installed: $(command -v git-secrets >/dev/null && echo YES || echo NO)"

echo ""
echo "=== Audit complete ==="
```

## Frequently Asked Questions

**How do I prioritize which recommendations to implement first?**

Start with changes that require the least effort but deliver the most impact. Quick wins build momentum and demonstrate value to stakeholders. Save larger structural changes for after you have established a baseline and can measure improvement.

**Do these recommendations work for small teams?**

Yes, most practices scale down well. Small teams can often implement changes faster because there are fewer people to coordinate. Adapt the specifics to your team size—a 5-person team does not need the same formal processes as a 50-person organization.

**How do I measure whether these changes are working?**

Define 2-3 measurable outcomes before you start. Track them weekly for at least a month to see trends. Common metrics include response time, completion rate, team satisfaction scores, and error frequency. Avoid measuring too many things at once.

**How do I handle team members in very different time zones?**

Establish a shared overlap window of at least 2-3 hours for synchronous work. Use async communication tools for everything else. Document decisions in writing so people in other time zones can catch up without needing a live recap.

**What is the biggest mistake people make when applying these practices?**

Trying to change everything at once. Pick one or two practices, implement them well, and let the team adjust before adding more. Gradual adoption sticks better than wholesale transformation, which often overwhelms people and gets abandoned.

## Related Articles

- [Linux Server Hardening Guide for Remote Developers](/remote-work-tools/linux-server-hardening-remote-developers/)
- [Best SSH Key Management Solution for Distributed Remote](/remote-work-tools/best-ssh-key-management-solution-for-distributed-remote-engi/)
- [VS Code Remote Development Setup Guide](/remote-work-tools/vscode-remote-development-setup/)
- [SSH Tunnels for Remote Database Access](/remote-work-tools/ssh-tunnels-remote-database-access/)
- [Security Tools for a Fully Remote Company Under 20 Employees](/remote-work-tools/security-tools-for-a-fully-remote-company-under-20-employees/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
