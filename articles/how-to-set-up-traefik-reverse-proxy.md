---
layout: default
title: "How to Set Up Traefik Reverse Proxy"
description: "Deploy Traefik as a reverse proxy with automatic SSL, Docker service discovery, and dashboard access for remote infrastructure teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-traefik-reverse-proxy/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Traefik is a reverse proxy that integrates natively with Docker and Kubernetes. Add labels to your Docker containers and Traefik automatically detects them, configures routing, and issues SSL certificates via Let's Encrypt — no nginx config files to write per service. For remote teams managing multiple services, it dramatically reduces the operational surface area.

---

## Deploy Traefik with Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/traefik.yml:/traefik.yml:ro
      - ./traefik/config:/config:ro
      - traefik-certs:/certs
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      # Dashboard
      - "traefik.http.routers.traefik.rule=Host(`traefik.yourcompany.com`)"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."

networks:
  proxy:
    external: true

volumes:
  traefik-certs:
```

Create the Docker network first:

```bash
docker network create proxy
```

---

## Traefik Static Configuration

```yaml
# traefik/traefik.yml
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true
  debug: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt

certificatesResolvers:
  letsencrypt:
    acme:
      email: ops@yourcompany.com
      storage: /certs/acme.json
      # Use tlsChallenge for simple setups
      tlsChallenge: {}
      # Or httpChallenge:
      # httpChallenge:
      #   entryPoint: web

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: proxy
  file:
    directory: /config
    watch: true

log:
  level: INFO
  format: json

accessLog:
  format: json
  fields:
    defaultMode: keep
    headers:
      defaultMode: drop
      names:
        User-Agent: keep
        X-Forwarded-For: keep
```

---

## Expose a Service with Labels

Any Docker container can be exposed through Traefik with labels:

```yaml
# your-app/docker-compose.yml
version: "3.8"

services:
  app:
    image: yourcompany/app:latest
    restart: unless-stopped
    networks:
      - proxy
      - internal
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.yourcompany.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      - "traefik.http.services.app.loadbalancer.server.port=3000"
    environment:
      - DATABASE_URL=postgresql://...

networks:
  proxy:
    external: true
  internal:
```

That's all that's needed. Traefik detects the container, acquires an SSL cert, and starts routing `app.yourcompany.com` to the container — no nginx config, no cert management.

---

## Dynamic Configuration for Non-Docker Services

For services not running in Docker (external APIs, bare-metal services), use file-based config:

```yaml
# traefik/config/external-services.yml
http:
  routers:
    legacy-api:
      rule: "Host(`api.yourcompany.com`)"
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
      service: legacy-api-service
      middlewares:
        - rate-limit
        - secure-headers

    internal-dashboard:
      rule: "Host(`metrics.yourcompany.com`)"
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
      service: grafana
      middlewares:
        - auth

  services:
    legacy-api-service:
      loadBalancer:
        servers:
          - url: "http://10.0.1.50:8080"
        healthCheck:
          path: /health
          interval: 30s
          timeout: 5s

    grafana:
      loadBalancer:
        servers:
          - url: "http://10.0.1.51:3000"

  middlewares:
    secure-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"

    rate-limit:
      rateLimit:
        average: 100
        burst: 50

    auth:
      basicAuth:
        usersFile: /config/htpasswd
```

---

## Wildcard Certificates with DNS Challenge

For `*.yourcompany.com` wildcard certs, use a DNS challenge:

```yaml
# traefik/traefik.yml additions
certificatesResolvers:
  letsencrypt-wildcard:
    acme:
      email: ops@yourcompany.com
      storage: /certs/acme-wildcard.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

Set environment variables for the Traefik container:

```yaml
# In docker-compose.yml environment section
environment:
  - CF_DNS_API_TOKEN=${CLOUDFLARE_DNS_TOKEN}
```

Use the wildcard cert in a router:

```yaml
labels:
  - "traefik.http.routers.app.tls.domains[0].main=yourcompany.com"
  - "traefik.http.routers.app.tls.domains[0].sans=*.yourcompany.com"
  - "traefik.http.routers.app.tls.certresolver=letsencrypt-wildcard"
```

---

## Load Balancing Multiple Replicas

Traefik automatically load balances when multiple containers with the same labels are running:

```bash
# Scale to 3 replicas — Traefik detects and round-robins automatically
docker-compose up --scale app=3 -d
```

For sticky sessions (session affinity):

```yaml
labels:
  - "traefik.http.services.app.loadbalancer.sticky.cookie=true"
  - "traefik.http.services.app.loadbalancer.sticky.cookie.name=lb_session"
  - "traefik.http.services.app.loadbalancer.sticky.cookie.secure=true"
```

---

## IP Allowlisting for Internal Services

Restrict access to internal dashboards by IP:

```yaml
# traefik/config/middlewares.yml
http:
  middlewares:
    office-only:
      ipAllowList:
        sourceRange:
          - "10.0.0.0/8"       # Internal network
          - "203.0.113.0/24"   # Office IP range
          - "198.51.100.42/32" # VPN exit IP
```

Apply to a router:

```yaml
labels:
  - "traefik.http.routers.internal-app.middlewares=office-only"
```

---

## Monitor with Traefik Access Logs

Parse access logs for slow requests:

```bash
# Find requests over 1 second
docker logs traefik 2>&1 \
  | jq 'select(.Duration > 1000000000)' \
  | jq '{url: .RequestPath, duration_ms: (.Duration/1000000), status: .DownstreamStatus}'
```

Check cert expiry dates:

```bash
curl -s "http://localhost:8080/api/overview" \
  | jq '.http.routers | to_entries[] | {name: .key, tls: .value.tls}'
```

---

## Related Reading

- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [How to Automate SSL Certificate Renewal](/remote-work-tools/how-to-automate-ssl-certificate-renewal/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Set Freelance Developer Rates in 2026](/remote-work-tools/how-to-set-freelance-developer-rates-2026/)

---

## Related Articles

- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [Setting Up Keycloak for Team SSO](/remote-work-tools/setting-up-keycloak-for-team-sso/)
- [Optimize Docker for Slow Connections When Working Remotely](/remote-work-tools/docker-optimize-slow-connection-remote-work/)
- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
