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

Unlike nginx, which requires you to write a new server block and reload config for every service you deploy, Traefik watches your Docker socket in real time. Deploy a new container with the right labels and traffic starts routing within seconds. Remove the container and routing disappears. This dynamic behavior is the core reason Traefik has become the default choice for teams running microservices or self-hosted tooling on a single host.

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

The `security_opt: no-new-privileges:true` line is important. It prevents the Traefik process from gaining additional privileges via setuid/setgid binaries — a defense-in-depth measure since Traefik mounts the Docker socket, which is a high-privilege resource. The Docker socket mount itself (`/var/run/docker.sock:ro`) is read-only to minimize the blast radius of any vulnerability in Traefik's Docker provider.

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

The `exposedByDefault: false` setting is critical for security. Without it, every Docker container is automatically exposed through Traefik as soon as it connects to the proxy network. With it set to false, only containers with the explicit label `traefik.enable=true` get routed. This prevents accidentally exposing internal databases, caches, or background workers that happen to share the network.

The `file` provider with `watch: true` means Traefik hot-reloads any YAML files you drop in `/config` without a restart. This is where you put routing rules for non-Docker services.

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

The dual-network pattern (proxy + internal) is intentional. The `proxy` network is the one Traefik watches. The `internal` network is for communication between your app container and its database or cache. Your database never touches the proxy network and therefore is never exposed to Traefik routing or the outside world.

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

The `healthCheck` on the legacy-api-service tells Traefik to probe `/health` every 30 seconds. If the probe fails, Traefik stops routing to that server and waits for it to recover before resuming. This prevents Traefik from sending traffic to a backend that is up at the TCP layer but not serving correctly.

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

Wildcard certs are particularly useful for internal tooling where you're adding new subdomains frequently. One cert covers all of them. The tradeoff is that the Cloudflare API token you provide needs DNS edit access — store it with care. Create a scoped API token in Cloudflare that has `Zone:DNS:Edit` access only to the relevant zone, rather than using your account-level API key.

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

Sticky sessions pin a user to the same backend container for the duration of their session. This matters for applications that store session state in memory rather than a shared store. The cleaner solution is moving session state to Redis and removing the need for stickiness, but sticky sessions are a valid interim step.

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

This pattern works well for locking down monitoring dashboards (Grafana, Prometheus, Alertmanager) and CI interfaces that should never be accessible from arbitrary internet addresses. Combine it with basic auth for defense in depth — IP allowlisting can be bypassed if someone is on a shared network or VPN.

For remote teams where developers connect from varying IP addresses, the VPN exit IP approach is common: all VPN traffic exits from a known static IP, and only that IP (plus any office ranges) is allowed.

---

## Health Checks and Circuit Breakers

Traefik supports health checks on backend services and can remove unhealthy servers from rotation automatically:

```yaml
# traefik/config/services.yml
http:
  services:
    app-service:
      loadBalancer:
        healthCheck:
          path: /healthz
          interval: 10s
          timeout: 3s
          # Remove server if it fails 3 consecutive checks
        servers:
          - url: "http://10.0.1.10:3000"
          - url: "http://10.0.1.11:3000"
```

For circuit breaking — stopping traffic to a service when its error rate spikes:

```yaml
http:
  middlewares:
    circuit-breaker:
      circuitBreaker:
        expression: "ResponseCodeRatio(500, 600, 0, 600) > 0.30 || NetworkErrorRatio() > 0.10"
```

This opens the circuit breaker when more than 30% of responses are 5xx errors or more than 10% of connections fail at the network level. During the open state, Traefik returns a 503 rather than forwarding requests to an overloaded backend.

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

List all active routers and their status via the API:

```bash
# View all HTTP routers
curl -s http://localhost:8080/api/http/routers | jq '.[].name'

# Check a specific router's configuration
curl -s http://localhost:8080/api/http/routers/app@docker | jq .

# See all services and their health
curl -s http://localhost:8080/api/http/services | jq '.[] | {name: .name, servers: .loadBalancer.servers}'
```

The Traefik dashboard at port 8080 (or behind your configured router) gives you a live view of all routers, services, and middleware — which containers are connected, which certs are active, and which routes are healthy. For remote teams where infrastructure changes happen asynchronously, the dashboard is the quickest way to verify a new service came up correctly without SSHing into the host.

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
