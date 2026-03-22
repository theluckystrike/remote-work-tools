---
layout: default
title: "How to Set Up Portainer for Docker Management"
description: "Deploy Portainer CE for visual Docker management, stack deployments, and remote access across multiple container hosts for distributed teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-portainer-for-docker-management/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Portainer gives remote teams a web UI for Docker and Kubernetes that replaces constant SSH sessions. Team members can view logs, restart containers, deploy stacks, and manage volumes without needing shell access or Docker CLI knowledge. Portainer CE is free and self-hostable.

---

## Deploy Portainer CE

Create a persistent volume and run the container:

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access at `https://your-server-ip:9443` on first boot and set your admin password.

Docker Compose version (recommended for production):

```yaml
# docker-compose.yml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - portainer_data:/data
    ports:
      - "9443:9443"
      - "8000:8000"

volumes:
  portainer_data:
```

---

## Nginx Reverse Proxy with SSL

Don't expose port 9443 directly. Proxy it through Nginx:

```nginx
# /etc/nginx/sites-available/portainer
server {
    listen 443 ssl http2;
    server_name portainer.yourcompany.com;

    ssl_certificate     /etc/letsencrypt/live/portainer.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/portainer.yourcompany.com/privkey.pem;

    # Portainer requires WebSocket support
    location / {
        proxy_pass          https://localhost:9443;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_set_header    Host $host;
        proxy_ssl_verify    off;  # Portainer uses self-signed cert internally
    }
}

server {
    listen 80;
    server_name portainer.yourcompany.com;
    return 301 https://$host$request_uri;
}
```

---

## Connect Remote Docker Hosts (Agents)

Portainer can manage multiple Docker hosts from one UI. Deploy the Portainer Agent on each remote host:

```bash
# On each remote Docker host
docker run -d \
  --name portainer_agent \
  --restart always \
  -p 9001:9001 \
  --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

Then in Portainer UI:
1. Go to **Environments > Add Environment**
2. Choose **Agent**
3. Enter the hostname: `remote-host.yourcompany.com:9001`
4. Name it (e.g., `prod-web-01`)

Now you can switch between hosts in the dropdown without leaving the UI.

For Swarm clusters:

```bash
# On the Swarm manager
docker service create \
  --name portainer_agent \
  --network portainer_agent_network \
  --mode global \
  --constraint 'node.platform.os == linux' \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=bind,src=/var/lib/docker/volumes,dst=/var/lib/docker/volumes \
  --mount type=bind,src=/,dst=/host \
  portainer/agent:latest
```

---

## Deploy Stacks via Portainer

Portainer can deploy Docker Compose stacks from:
- Paste YAML directly in the UI
- A Git repository (auto-deploy on push)
- A file upload

For Git-backed stacks (recommended for remote teams):

1. **Stacks > Add Stack > Repository**
2. Enter your repo URL, branch, and compose file path
3. Enable **GitOps updates** to auto-redeploy when the branch updates
4. Add any environment variables in the **Environment variables** section

This means any team member with Portainer access can trigger a redeploy by merging to the configured branch, without SSH access to the host.

---

## Environment Variables and Secrets

Store secrets as Portainer environment variables instead of hardcoding in compose files:

```yaml
# docker-compose.yml (reference env vars, don't hardcode)
version: "3.8"
services:
  app:
    image: yourcompany/app:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SECRET_KEY=${SECRET_KEY}
```

In Portainer when deploying the stack, add the actual values in the **Environment variables** section. They're stored encrypted and never appear in version control.

---

## Role-Based Access Control

Portainer CE supports basic RBAC. For remote teams:

1. Go to **Settings > Users > Add user**
2. Create users with roles:
 - **Administrator**: Full access
 - **Standard user**: Can deploy and manage in assigned environments
 - **Read-only user**: Can view logs and container state, cannot modify

Assign environments to teams:
1. **Settings > Teams > Add team**
2. Create teams matching your org structure (frontend, backend, devops)
3. Assign environments to teams under **Environments > Edit > Access**

This means the frontend team can manage their containers but can't touch the database host.

---

## Container Log Access for Remote Teams

One of Portainer's most useful features for remote teams is log access without SSH:

1. Navigate to **Environments > your-host > Containers**
2. Click any container name
3. Click **Logs** tab
4. Enable **Auto-refresh** to tail logs in real time
5. Use the search box to filter log lines

For debugging, the **Console** tab opens a terminal in the running container (equivalent to `docker exec -it container bash`) — without SSH access to the host.

---

## Backup Portainer Configuration

Portainer stores all configuration in the Docker volume `portainer_data`. Back it up:

```bash
#!/bin/bash
# scripts/backup-portainer.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/portainer"
mkdir -p "$BACKUP_DIR"

# Stop briefly for consistent backup (< 5 seconds)
docker stop portainer

docker run --rm \
  -v portainer_data:/data \
  -v "$BACKUP_DIR":/backup \
  alpine tar czf "/backup/portainer_data_$DATE.tar.gz" -C /data .

docker start portainer

# Keep only last 14 days
find "$BACKUP_DIR" -name "portainer_data_*.tar.gz" -mtime +14 -delete

echo "Backup complete: $BACKUP_DIR/portainer_data_$DATE.tar.gz"
```

Restore:

```bash
docker stop portainer
docker run --rm \
  -v portainer_data:/data \
  -v /backups/portainer:/backup \
  alpine tar xzf "/backup/portainer_data_20260322_120000.tar.gz" -C /data
docker start portainer
```

---

## Useful API Commands

Portainer has a REST API for automation:

```bash
# Authenticate
TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  https://portainer.yourcompany.com/api/auth \
  | jq -r '.jwt')

# List all containers on environment 1
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.yourcompany.com/api/endpoints/1/docker/containers/json \
  | jq '.[].Names'

# Restart a container
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.yourcompany.com/api/endpoints/1/docker/containers/myapp/restart
```

---

## Related Reading

- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Set Up Woodpecker CI for Self-Hosted](/remote-work-tools/how-to-set-up-woodpecker-ci-for-self-hosted/)
- [How to Set Up Traefik Reverse Proxy](/remote-work-tools/how-to-set-up-traefik-reverse-proxy/)

- [How to Set Up Ansible for Remote Server Management](/remote-work-tools/how-to-set-up-ansible-remote-server-management/)
---

## Related Articles

- [How to Set Up Traefik Reverse Proxy](/remote-work-tools/how-to-set-up-traefik-reverse-proxy/)
- [Setting Up Keycloak for Team SSO](/remote-work-tools/setting-up-keycloak-for-team-sso/)
- [Optimize Docker for Slow Connections When Working Remotely](/remote-work-tools/docker-optimize-slow-connection-remote-work/)
- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
