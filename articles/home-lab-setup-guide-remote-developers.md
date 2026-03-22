---
layout: default
title: "Home Lab Setup Guide for Remote Developers"
description: "Build a home lab for remote development: hardware selection, hypervisor setup, network segmentation, DNS, and services worth running locally for development"
date: 2026-03-21
author: theluckystrike
permalink: /home-lab-setup-guide-remote-developers/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

A home lab gives you a real infrastructure environment to experiment with, a place to run services locally for development, and a learning ground for infrastructure skills that are difficult to practice on cloud free tiers alone. For remote developers, it also means always-available compute and storage that you own.

## Table of Contents

- [Hardware: What to Buy in 2026](#hardware-what-to-buy-in-2026)
- [Hypervisor: Proxmox VE](#hypervisor-proxmox-ve)
- [Create Your First VM](#create-your-first-vm)
- [Network: VLANs for Isolation](#network-vlans-for-isolation)
- [DNS: pi-hole + Unbound](#dns-pi-hole-unbound)
- [Services Worth Running in a Home Lab](#services-worth-running-in-a-home-lab)
- [SSH Config for Lab Access](#ssh-config-for-lab-access)
- [Remote Access via Tailscale](#remote-access-via-tailscale)
- [Backups: The Step Most People Skip](#backups-the-step-most-people-skip)
- [Related Reading](#related-reading)

This guide covers: hardware choice, hypervisor installation, network setup, and the services worth running in a home lab for development work.

## Hardware: What to Buy in 2026

The sweet spot for a developer home lab is a small form factor PC or repurposed workstation. Avoid consumer NAS devices — they limit your software options.

**Recommended builds:**

| Use Case | Hardware | Cost |
|---|---|---|
| Minimal (VMs + services) | Beelink SEi12 i5 mini PC, 32GB RAM, 1TB SSD | ~$300 |
| Mid-tier (Kubernetes + CI/CD) | Intel NUC 13 Pro, 64GB RAM, 2TB NVMe | ~$600 |
| Full dev cluster | 2x HP EliteDesk 800 G3 (used), 32GB each | ~$250 total |
| Repurposed workstation | Used Lenovo ThinkStation P320, 64GB ECC RAM | ~$200 used |

Key specs to prioritize: RAM (you need at least 32GB for running multiple VMs), SSD storage (spinning disk kills VM performance), and CPU virtualization support (check with `grep -E 'vmx|svm' /proc/cpuinfo`).

Power consumption matters for always-on hardware. The Beelink mini PC draws around 15-25W under load — roughly $2-3/month in electricity at average US rates. Compare that to a full tower workstation at 150W+ idle, which runs $15-20/month continuously. For a 24/7 lab, mini PCs and NUCs win on running costs, and the noise level is also significantly lower — important if the lab lives in a home office or bedroom.

## Hypervisor: Proxmox VE

Proxmox is the standard home lab hypervisor. It runs KVM virtual machines and LXC containers, has a web UI, and is free with optional paid support.

```bash
# Download Proxmox VE ISO from proxmox.com
# Flash to USB
sudo dd if=proxmox-ve_8.2-1.iso of=/dev/sdX bs=4M status=progress

# Boot from USB and follow installer
# Set static IP during install: e.g., 192.168.1.100
# Access web UI at https://192.168.1.100:8006
```

After install, update and clean up the default enterprise repos:

```bash
# SSH into Proxmox host
ssh root@192.168.1.100

# Remove enterprise repo (requires paid subscription)
rm /etc/apt/sources.list.d/pve-enterprise.list

# Add free repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt dist-upgrade -y
```

**Alternatives to Proxmox:** If you prefer something lighter, consider:
- **XCP-ng** — another open-source KVM hypervisor, closer to VMware's interface
- **Incus** — LXC/VM management for those who prefer the LXD lineage without the Canonical dependency
- **libvirt + virt-manager** — bare metal KVM with a desktop GUI, ideal if you prefer managing VMs from a Linux workstation

For most developers, Proxmox hits the best balance of features and operational simplicity.

## Create Your First VM

```bash
# Via CLI (or use the web UI)
# Download Ubuntu 24.04 cloud image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img \
  -P /var/lib/vz/template/iso/

# Create VM
qm create 100 \
  --name ubuntu-dev \
  --memory 4096 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img \
  --ide2 local-lvm:cloudinit \
  --boot order=scsi0 \
  --serial0 socket --vga serial0

# Set cloud-init options
qm set 100 \
  --ciuser devuser \
  --sshkeys ~/.ssh/authorized_keys \
  --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1

# Start VM
qm start 100

# SSH in
ssh devuser@192.168.1.101
```

Cloud-init VMs boot with your SSH key already installed — no password needed.

## Network: VLANs for Isolation

Keep lab traffic separate from your home network. Most managed switches (TP-Link TL-SG108E, ~$30) support VLANs.

```
VLAN 1 (untagged): home network — laptops, phones
VLAN 10: lab management — Proxmox web UI access
VLAN 20: lab services — VMs, containers
VLAN 30: IoT (optional)
```

In Proxmox, add a VLAN bridge:

```bash
# /etc/network/interfaces — add VLAN-aware bridge
auto vmbr0
iface vmbr0 inet static
        address 192.168.1.100/24
        gateway 192.168.1.1
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094
```

VLAN isolation provides two practical benefits for developers: your lab experiments cannot accidentally DDoS your home router, and you can simulate realistic network topologies (frontend subnet, backend subnet, database subnet) without physical hardware.

## DNS: pi-hole + Unbound

Pi-hole handles ad blocking and local DNS resolution. Unbound adds a recursive resolver so DNS queries go directly to root nameservers — not Google or Cloudflare.

```bash
# Install Pi-hole in an LXC container
# From Proxmox shell:
pct create 200 \
  local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst \
  --hostname pihole \
  --memory 512 \
  --cores 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1 \
  --start 1

pct exec 200 -- bash -c "$(curl -fsSL https://install.pi-hole.net)"

# Install Unbound in the same container
pct exec 200 -- apt install unbound -y

# Add local DNS records for lab hosts
# /etc/pihole/custom.list
192.168.1.100  proxmox.lab
192.168.1.101  ubuntu-dev.lab
192.168.1.200  pihole.lab
```

Set your router's DHCP to push 192.168.1.200 as the DNS server.

Local DNS entries are a quality-of-life improvement that compounds over time. Typing `ssh ubuntu-dev.lab` instead of memorizing IP addresses, and accessing services at `gitea.lab:3000` instead of `192.168.1.101:3000`, makes the lab feel like a real infrastructure environment.

## Services Worth Running in a Home Lab

### Gitea (self-hosted Git)

```bash
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 2222:22 \
  -v /opt/gitea:/data \
  -e USER_UID=1000 \
  -e USER_GID=1000 \
  gitea/gitea:1.21
```

Use Gitea as a local mirror of your GitHub repos. Push to local first, then to GitHub — useful when GitHub is down or for private experimentation. Gitea also supports CI integration via Woodpecker, which is covered below.

### Registry (Docker image cache)

```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v /opt/registry:/var/lib/registry \
  registry:2
```

Pull images locally and push to your private registry. Reference as `192.168.1.101:5000/yourimage:tag`. Dramatically faster than pulling from Docker Hub on subsequent `docker pull` operations.

### Minio (S3-compatible storage)

```bash
docker run -d \
  --name minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -v /opt/minio:/data \
  -e MINIO_ROOT_USER=admin \
  -e MINIO_ROOT_PASSWORD=changeme123 \
  quay.io/minio/minio server /data --console-address ":9001"
```

Test S3 code locally without AWS charges. The AWS SDK works against Minio by setting `endpoint_url`.

### Woodpecker CI (self-hosted CI/CD)

A local CI runner eliminates GitHub Actions minute limits during heavy development cycles. Woodpecker CI is a lightweight, Docker-native CI system that uses the same YAML pipeline format as Drone CI:

```bash
# docker-compose.yml for Woodpecker
version: '3'
services:
  woodpecker-server:
    image: woodpeckerci/woodpecker-server:latest
    ports:
      - 8000:8000
    environment:
      - WOODPECKER_OPEN=true
      - WOODPECKER_GITEA=true
      - WOODPECKER_GITEA_URL=http://192.168.1.101:3000
      - WOODPECKER_AGENT_SECRET=supersecret

  woodpecker-agent:
    image: woodpeckerci/woodpecker-agent:latest
    environment:
      - WOODPECKER_SERVER=woodpecker-server:9000
      - WOODPECKER_AGENT_SECRET=supersecret
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Pair Woodpecker with your Gitea instance for a fully local Git + CI pipeline. This is particularly useful for validating Docker build pipelines and infrastructure-as-code changes before pushing to production.

## SSH Config for Lab Access

```bash
# ~/.ssh/config
Host proxmox
  HostName 192.168.1.100
  User root
  IdentityFile ~/.ssh/id_homelab

Host ubuntu-dev
  HostName 192.168.1.101
  User devuser
  IdentityFile ~/.ssh/id_homelab
  ProxyJump proxmox

# Access from outside home network via Tailscale
Host lab-dev
  HostName 100.64.0.5  # Tailscale IP of ubuntu-dev
  User devuser
  IdentityFile ~/.ssh/id_homelab
```

## Remote Access via Tailscale

```bash
# Install Tailscale on Proxmox host and key VMs
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-routes=192.168.1.0/24 --accept-routes

# On Proxmox host, enable IP forwarding
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p

# In Tailscale admin console, approve the route advertised above
# Now your laptop can reach all lab IPs via Tailscale from anywhere
```

With the subnet route approved, your work laptop reaches `192.168.1.101` through the Tailscale tunnel from any network — coffee shop, co-working space, hotel. This is the key feature that makes a home lab useful for remote developers rather than just a home project.

Tailscale's free tier supports up to 3 users and 100 devices, more than enough for a personal lab. The Magic DNS feature (tailscale.net hostnames) adds another layer of naming convenience on top of your Pi-hole local DNS.

## Backups: The Step Most People Skip

A home lab without backups is a lab you will eventually rebuild from scratch. Proxmox Backup Server (PBS) is free and designed for this use case:

```bash
# Install PBS on a second machine or separate VM
# Then configure a backup job in Proxmox web UI:
# Datacenter > Backup > Add
# Schedule: daily at 02:00
# Mode: Snapshot (no downtime)
# Retention: 7 daily, 4 weekly
```

For offsite backup, Restic against a Backblaze B2 bucket costs roughly $0.006/GB/month. A 500GB backup set costs about $3/month — worth it to protect weeks of configuration work.

## Related Reading

- [Prometheus Monitoring Setup for Remote Infrastructure](/remote-work-tools/prometheus-monitoring-remote-infrastructure/)
- [How to Set Up WireGuard VPN Server for Small Remote Development Teams](/remote-work-tools/how-to-set-up-wireguard-vpn-server-for-small-remote-developm/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
- [Redshift - Linux/Unix blue light filter](/remote-work-tools/best-home-office-setup-for-software-developers/)

## Related Articles

- [Remote Work VoIP Setup for Home Offices](/remote-work-tools/remote-work-voip-setup-for-home-offices/)
- [How to Set Up Home Office Network for Remote Work](/remote-work-tools/how-to-set-up-home-office-network-for-remote-work/)
- [VS Code Remote Development Setup Guide](/remote-work-tools/vscode-remote-development-setup/)
- [How to Set Up HIPAA Compliant Home Office for Remote](/remote-work-tools/how-to-set-up-hipaa-compliant-home-office-for-remote-healthc/)
- [Best Webcam for Home Office Remote Work: A Technical Guide](/remote-work-tools/best-webcam-for-home-office-remote-work/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**How long does it take to guide for remote developers?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Can I adapt this for a different tech stack?**

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

{% endraw %}
