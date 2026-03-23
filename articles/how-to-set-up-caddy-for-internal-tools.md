---
layout: default
title: "How to Set Up Caddy for Internal Tools"
description: "Deploy Caddy as a reverse proxy for internal remote team tools with automatic HTTPS, auth middleware, and zero-downtime reloads"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-caddy-for-internal-tools/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Caddy is the easiest way to put HTTPS in front of every internal tool your team runs. Grafana, Kibana, Gitea, a private Retool instance, whatever. It auto-provisions TLS from Let's Encrypt, reloads config without dropping connections, and ships a dead-simple declarative config format. This guide walks through a production-ready Caddy setup for a distributed team.

Why Caddy Over nginx or Traefik

nginx requires manual cert renewal via cron + certbot. Traefik is great but demands a running Docker daemon and a non-trivial label schema. Caddy gives you automatic HTTPS with zero cron jobs, a single Caddyfile, and a JSON API for dynamic updates. ideal for small platform teams.

Caddy's hard advantages:
- `tls internal` generates a local CA and signs certs for `.internal` domains with no DNS challenge
- `reload` is atomic. workers drain gracefully before config swap
- Built-in basic auth, rate limiting, request logging, and header manipulation
- Single ~50 MB binary, no external dependencies

Install Caddy

```bash
Debian/Ubuntu
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy

macOS (Homebrew)
brew install caddy

Binary download (any Linux)
curl -L "https://github.com/caddyserver/caddy/releases/latest/download/caddy_linux_amd64.tar.gz" \
  | tar -xz caddy
sudo mv caddy /usr/local/bin/
sudo setcap CAP_NET_BIND_SERVICE=+eip /usr/local/bin/caddy
```

Verify:

```bash
caddy version
v2.9.1 h1:...
```

Directory Layout

```
/etc/caddy/
 Caddyfile          # main config
 snippets/
    auth.caddy     # shared auth block
    headers.caddy  # shared security headers
 tls/
     cert.pem       # optional manual cert
     key.pem
```

Create directories:

```bash
sudo mkdir -p /etc/caddy/snippets /etc/caddy/tls
sudo chown -R caddy:caddy /etc/caddy
```

Basic Caddyfile for Internal Tools

```caddyfile
/etc/caddy/Caddyfile

{
    # Global options
    admin 127.0.0.1:2019        # JSON API. never expose publicly
    log {
        level  INFO
        format json
    }
    # Use internal CA for .internal domains
    local_certs
}

Grafana
grafana.internal.example.com {
    reverse_proxy localhost:3000
    import snippets/auth.caddy
    import snippets/headers.caddy
    tls internal
}

Kibana
kibana.internal.example.com {
    reverse_proxy localhost:5601
    import snippets/auth.caddy
    import snippets/headers.caddy
    tls internal
}

Gitea
git.internal.example.com {
    reverse_proxy localhost:3001 {
        header_up X-Forwarded-Proto {scheme}
        header_up X-Real-IP {remote_host}
    }
    import snippets/headers.caddy
    tls internal
}

Retool / internal app
app.internal.example.com {
    reverse_proxy localhost:3002
    import snippets/auth.caddy
    import snippets/headers.caddy
    tls internal
}
```

auth.caddy Snippet

```caddyfile
/etc/caddy/snippets/auth.caddy
Protects any site with HTTP basic auth
basicauth {
    # Generate hash: caddy hash-password --plaintext "your-password"
    ops       $2a$14$K6GzFsRqZZqBxl3GNXR8i.D7ZW9IuJLiNFwqPCl5w0xE3h3M4pBfG
    devteam   $2a$14$T9wAz8kXVs1N2mLqY3dBjuR5HqPu4nZ1cF7bEeRdSm0vW8kP3oAqC
}
```

Generate a new hash:

```bash
caddy hash-password --plaintext "your-secure-password"
```

headers.caddy Snippet

```caddyfile
/etc/caddy/snippets/headers.caddy
header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains"
    X-Content-Type-Options    "nosniff"
    X-Frame-Options           "SAMEORIGIN"
    Referrer-Policy           "strict-origin-when-cross-origin"
    -Server
    -X-Powered-By
}
```

Wildcard TLS With DNS Challenge

For a real domain using AWS Route 53:

```bash
Install the Route53 DNS plugin
xcaddy build --with github.com/caddy-dns/route53
```

```caddyfile
{
    admin 127.0.0.1:2019
    acme_dns route53 {
        access_key_id     {$AWS_ACCESS_KEY_ID}
        secret_access_key {$AWS_SECRET_ACCESS_KEY}
        region            us-east-1
    }
}

*.internal.example.com {
    tls {
        dns route53
    }

    @grafana  host grafana.internal.example.com
    @kibana   host kibana.internal.example.com
    @git      host git.internal.example.com

    handle @grafana {
        reverse_proxy localhost:3000
        import snippets/auth.caddy
    }
    handle @kibana {
        reverse_proxy localhost:5601
        import snippets/auth.caddy
    }
    handle @git {
        reverse_proxy localhost:3001
    }

    import snippets/headers.caddy
}
```

Systemd Service

If you installed from the apt repo, systemd is already configured. For a binary install:

```bash
/etc/systemd/system/caddy.service
sudo tee /etc/systemd/system/caddy.service > /dev/null <<'EOF'
[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/local/bin/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/usr/local/bin/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now caddy
```

Zero-Downtime Config Reload

```bash
Validate config before reloading
caddy validate --config /etc/caddy/Caddyfile

Reload running instance (no dropped connections)
sudo systemctl reload caddy

Or use the API directly
curl -X POST "http://127.0.0.1:2019/load" \
  -H "Content-Type: text/caddyfile" \
  --data-binary @/etc/caddy/Caddyfile
```

Rate Limiting and IP Allow Lists

```caddyfile
grafana.internal.example.com {
    # Only allow VPN subnet
    @allowed remote_ip 10.8.0.0/16 192.168.1.0/24
    handle @allowed {
        reverse_proxy localhost:3000
    }
    respond "Forbidden" 403

    # Rate limit: 100 requests/minute per IP
    rate_limit {remote_host} 100r/m
}
```

OAuth2 Proxy Integration

For SSO via GitHub/Google, run `oauth2-proxy` alongside Caddy:

```caddyfile
grafana.internal.example.com {
    # Forward auth to oauth2-proxy
    forward_auth localhost:4180 {
        uri /oauth2/auth
        copy_headers X-Auth-Request-User X-Auth-Request-Email
    }
    reverse_proxy localhost:3000
    import snippets/headers.caddy
    tls internal
}
```

Start oauth2-proxy:

```bash
oauth2-proxy \
  --provider=github \
  --client-id="${GITHUB_CLIENT_ID}" \
  --client-secret="${GITHUB_CLIENT_SECRET}" \
  --github-org=your-org \
  --cookie-secret="$(openssl rand -base64 32)" \
  --email-domain="*" \
  --upstream=http://localhost:3000 \
  --http-address=127.0.0.1:4180
```

Logging to a File

```caddyfile
grafana.internal.example.com {
    log {
        output file /var/log/caddy/grafana-access.log {
            roll_size     100MiB
            roll_keep     10
            roll_keep_for 720h
        }
        format json
        level  INFO
    }
    reverse_proxy localhost:3000
}
```

Health Check Endpoint

```caddyfile
:2020 {
    respond /health 200
}
```

Verify from monitoring:

```bash
curl -sf http://localhost:2020/health && echo "Caddy UP"
```

Troubleshooting

| Problem | Check |
|---|---|
| `permission denied` binding port 80/443 | `sudo setcap CAP_NET_BIND_SERVICE=+eip /usr/local/bin/caddy` |
| TLS cert not provisioned | `journalctl -u caddy -f`. look for ACME errors |
| 502 Bad Gateway | Is the upstream actually running on the expected port? |
| Config reload fails | `caddy validate` first. parse errors are printed clearly |
| `tls internal` not trusted by browser | Install Caddy's local CA: `caddy trust` |

Install the local CA on macOS clients so browsers trust `.internal` certs:

```bash
On the Caddy server
sudo caddy trust

Copy the root CA to clients
scp caddy-host:/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt ~/caddy-local.crt

macOS
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/caddy-local.crt
```

Related Reading

- [ArgoCD GitOps Workflow Setup](/argocd-gitops-workflow-setup/)
- [How to Set Up Flux CD for GitOps](/how-to-set-up-flux-cd-for-gitops/)
- [How to Set Up Backstage Developer Portal](/how-to-set-up-backstage-developer-portal/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
