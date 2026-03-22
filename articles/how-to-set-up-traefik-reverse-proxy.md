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

## Middleware Chains for Production Hardening

Combining multiple middlewares into a chain gives you layered defense — rate limiting, secure headers, and auth in one pass. Define the chain in your dynamic config, then apply the chain name to any router.

```yaml
# traefik/config/middleware-chains.yml
http:
  middlewares:
    # Individual middlewares
    rate-limit-api:
      rateLimit:
        average: 60
        burst: 20
        period: 1m
        sourceCriterion:
          ipStrategy:
            depth: 1

    compress:
      compress:
        excludedContentTypes:
          - text/event-stream

    secure-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        frameDeny: true
        forceSTSHeader: true
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        referrerPolicy: "strict-origin-when-cross-origin"
        permissionsPolicy: "camera=(), microphone=(), geolocation=()"
        customResponseHeaders:
          X-Robots-Tag: "noindex, nofollow"  # for internal services

    # Chain them together
    api-chain:
      chain:
        middlewares:
          - rate-limit-api
          - secure-headers
          - compress

    internal-chain:
      chain:
        middlewares:
          - office-only
          - secure-headers
```

Apply a chain to a container with one label:

```yaml
labels:
  - "traefik.http.routers.api.middlewares=api-chain@file"
```

The `@file` suffix tells Traefik the middleware is defined in a file provider, not Docker labels. Chains are reusable — define once, apply to any router.

---

## Observability: Metrics and Tracing

Traefik exposes Prometheus metrics natively. Add the metrics endpoint to `traefik.yml`:

```yaml
# traefik/traefik.yml additions
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0
    entryPoint: metrics

entryPoints:
  metrics:
    address: ":8082"
```

Scrape from Prometheus with:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: traefik
    static_configs:
      - targets: ["traefik:8082"]
    metrics_path: /metrics
```

Key metrics to alert on:

| Metric | What to watch |
|--------|---------------|
| `traefik_entrypoint_requests_total` | Request volume by status code |
| `traefik_entrypoint_request_duration_seconds` | p99 latency per entrypoint |
| `traefik_service_open_connections` | Connection saturation |
| `traefik_router_requests_total{code="502"}` | Upstream failures |

For distributed tracing, add OpenTelemetry export (Traefik v3+):

```yaml
tracing:
  otlp:
    grpc:
      endpoint: tempo.internal:4317
      insecure: true
```

This sends trace spans to Grafana Tempo or any OTLP-compatible backend, correlating Traefik routing decisions with downstream service spans.

---

## Troubleshooting Common Traefik Issues

**Certificate not renewing:** Check `docker logs traefik` for ACME errors. Common causes: the domain doesn't resolve to this server (Let's Encrypt can't complete the challenge), or `acme.json` has wrong permissions (`chmod 600 acme.json`). For DNS challenge failures, confirm the API token has zone edit permissions.

**Service returns 502 Bad Gateway:** Traefik reached the container but the container rejected the connection. Verify the `loadbalancer.server.port` label matches the actual port your app listens on. Check `docker inspect <container>` to confirm the container is on the `proxy` network.

**Redirect loop on HTTPS:** If the upstream service also redirects HTTP→HTTPS, and Traefik forwards to it via HTTP internally, you get a loop. Fix: ensure the upstream app trusts `X-Forwarded-Proto` and only redirects when it's missing, or connect Traefik to the service via HTTPS with `--serversTransport.insecureSkipVerify=true` (dev only).

**Dashboard not loading:** The API router requires the `api@internal` service and must be on the `websecure` entrypoint. Confirm `api.dashboard: true` is in `traefik.yml` and your router labels include `traefik.http.routers.traefik.service=api@internal`.

**New container not discovered:** Ensure the container is on the `proxy` network (not just `bridge`) and has `traefik.enable=true`. Run `docker network inspect proxy` to confirm the container appears. If you added the container after Traefik started, Traefik should detect it automatically within seconds — check logs for `"Skipping provider"` messages.

```bash
# Inspect what Traefik currently sees
curl http://localhost:8080/api/rawdata | jq '.routers | keys'

# Watch for configuration events in real time
docker logs -f traefik 2>&1 | grep -E "(error|warn|router|service)"
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
