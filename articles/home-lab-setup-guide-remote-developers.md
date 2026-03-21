---
layout: default
title: "Home Lab Setup Guide for Remote Developers"
description: "Build a home lab for remote development: hardware selection, hypervisor setup, network segmentation, DNS, and services worth running locally for development work."
date: 2026-03-21
author: theluckystrike
permalink: /home-lab-setup-guide-remote-developers/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A home lab gives you a real infrastructure environment to experiment with, a place to run services locally for development, and a learning ground for infrastructure skills that are difficult to practice on cloud free tiers alone. For remote developers, it also means always-available compute and storage that you own.

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

Use Gitea as a local mirror of your GitHub repos. Push to local first, then to GitHub — useful when GitHub is down or for private experimentation.

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

With the subnet route approved, your work laptop reaches `192.168.1.101` through the Tailscale tunnel from any network — coffee shop, co-working space, hotel.

## Related Reading

- [Prometheus Monitoring Setup for Remote Infrastructure](/remote-work-tools/prometheus-monitoring-remote-infrastructure/)
- [How to Set Up WireGuard VPN Server for Small Remote Development Teams](/remote-work-tools/how-to-set-up-wireguard-vpn-server-for-small-remote-developm/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
