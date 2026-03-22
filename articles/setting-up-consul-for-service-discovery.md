---
layout: default
title: "Setting Up Consul for Service Discovery"
description: "Deploy HashiCorp Consul for service discovery, health checking, and dynamic configuration across distributed remote team infrastructure"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-consul-for-service-discovery/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Hardcoded service addresses are a maintenance nightmare as infrastructure scales. Consul provides service discovery (services register themselves and find each other by name), health checks (unhealthy instances are automatically removed from the pool), and a distributed key-value store for dynamic configuration. For remote teams, it replaces a spreadsheet of service addresses with a live, self-maintaining registry.

---

## Architecture

Consul runs in two modes:
- **Server**: Maintains the cluster state. Run 3 or 5 for fault tolerance.
- **Client/Agent**: Runs on every host, registers services, proxies requests to servers.

Minimum production setup: 3 Consul servers + Consul agents on each application host.

---

## Deploy with Docker Compose (Development/Single-Node)

```yaml
# docker-compose.yml
version: "3.8"
services:
  consul-server:
    image: hashicorp/consul:1.18
    container_name: consul
    restart: unless-stopped
    ports:
      - "8500:8500"   # HTTP API and UI
      - "8600:8600/udp"  # DNS
      - "8300:8300"   # Server RPC
    volumes:
      - consul-data:/consul/data
      - ./consul/config:/consul/config
    command: >
      agent
      -server
      -bootstrap-expect=1
      -ui
      -client=0.0.0.0
      -bind=0.0.0.0
      -advertise=YOUR_HOST_IP
      -node=consul-server-1
      -datacenter=dc1
      -data-dir=/consul/data
      -config-dir=/consul/config
    networks:
      - consul-net

volumes:
  consul-data:

networks:
  consul-net:
    driver: bridge
```

---

## Production 3-Node Cluster

On each Consul server node, use this config:

```json
// /etc/consul.d/consul.hcl
{
  "datacenter": "dc1",
  "data_dir": "/opt/consul/data",
  "log_level": "INFO",
  "server": true,
  "bootstrap_expect": 3,
  "bind_addr": "0.0.0.0",
  "advertise_addr": "{{ GetPrivateInterfaces | include \"network\" \"10.0.0.0/8\" | attr \"address\" }}",
  "client_addr": "0.0.0.0",
  "retry_join": [
    "10.0.1.10",
    "10.0.1.11",
    "10.0.1.12"
  ],
  "ui_config": {
    "enabled": true
  },
  "telemetry": {
    "prometheus_retention_time": "60s"
  },
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "enable_token_persistence": true
  },
  "verify_incoming": true,
  "verify_outgoing": true,
  "verify_server_hostname": true,
  "ca_file": "/etc/consul.d/tls/ca.pem",
  "cert_file": "/etc/consul.d/tls/server.pem",
  "key_file": "/etc/consul.d/tls/server-key.pem"
}
```

Generate TLS certificates:

```bash
# Install consul CLI
brew install consul

# Generate CA
consul tls ca create

# Generate server certs (run for each server)
consul tls cert create -server -dc dc1

# Generate client certs
consul tls cert create -client
```

Start Consul:

```bash
systemctl enable consul
systemctl start consul

# Verify cluster formed
consul members
```

---

## Register Services

Each application registers itself with Consul. For a Docker service:

```json
// /etc/consul.d/payments-service.json
{
  "service": {
    "id": "payments-service-01",
    "name": "payments-service",
    "tags": ["v2", "production"],
    "address": "10.0.1.20",
    "port": 8080,
    "meta": {
      "version": "v2.1.0",
      "region": "us-east"
    },
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s",
      "timeout": "3s",
      "deregister_critical_service_after": "60s"
    }
  }
}
```

Register via API (for containers that can't write files):

```bash
curl -X PUT \
  -H "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
  http://consul.yourcompany.internal:8500/v1/agent/service/register \
  -d '{
    "Name": "payments-service",
    "ID": "payments-service-'$(hostname)'",
    "Address": "'$(hostname -I | awk '{print $1}')'",
    "Port": 8080,
    "Tags": ["production"],
    "Check": {
      "HTTP": "http://localhost:8080/health",
      "Interval": "10s",
      "Timeout": "3s"
    }
  }'
```

Deregister on container stop:

```bash
# In container stop/entrypoint script
trap 'consul services deregister -id="payments-service-$(hostname)"' TERM
```

---

## Service Discovery in Applications

**DNS-based discovery (simplest):**

```bash
# Any service registered as "payments-service" is discoverable at:
# payments-service.service.consul

# Test:
dig @127.0.0.1 -p 8600 payments-service.service.consul

# Point your app at the Consul DNS resolver:
export PAYMENTS_API_URL="http://payments-service.service.consul:8080"
```

Configure systemd-resolved to forward `.consul` queries:

```ini
# /etc/systemd/resolved.conf.d/consul.conf
[Resolve]
DNS=127.0.0.1:8600
Domains=~consul
```

**HTTP API discovery (Go example):**

```go
import (
    "github.com/hashicorp/consul/api"
)

func discoverService(name string) (string, error) {
    client, err := api.NewDefaultClient()
    if err != nil {
        return "", err
    }

    services, _, err := client.Health().Service(
        name,
        "",
        true,  // Only passing health checks
        nil,
    )
    if err != nil {
        return "", err
    }

    if len(services) == 0 {
        return "", fmt.Errorf("no healthy instances of %s found", name)
    }

    // Round-robin: pick a random healthy instance
    instance := services[rand.Intn(len(services))]
    addr := fmt.Sprintf("http://%s:%d",
        instance.Service.Address,
        instance.Service.Port,
    )
    return addr, nil
}
```

---

## Key-Value Store for Dynamic Config

Consul's KV store replaces configuration that would otherwise require redeployments:

```bash
# Write configuration values
consul kv put config/payments-service/db-pool-size 25
consul kv put config/payments-service/rate-limit-per-user 100
consul kv put config/payments-service/feature-new-checkout true

# Read
consul kv get config/payments-service/db-pool-size
```

Watch for changes and reload app config:

```bash
# Watch a key and trigger action on change
consul watch \
  -type=key \
  -key=config/payments-service/rate-limit-per-user \
  /opt/scripts/reload-config.sh
```

```python
# Python: watch KV and update in-memory config
import consul
import threading

c = consul.Consul(host='consul.yourcompany.internal')

def watch_config():
    index = None
    while True:
        index, data = c.kv.get(
            'config/payments-service/rate-limit-per-user',
            index=index,
            wait='5m'
        )
        if data:
            new_limit = int(data['Value'].decode())
            app_config['rate_limit'] = new_limit
            logging.info(f"Rate limit updated to {new_limit}")

threading.Thread(target=watch_config, daemon=True).start()
```

---

## Health Check Best Practices

```json
{
  "check": {
    "id": "payments-http",
    "name": "HTTP API health",
    "http": "http://localhost:8080/health",
    "interval": "10s",
    "timeout": "3s",
    "deregister_critical_service_after": "60s",
    "success_before_passing": 2,    // Must pass 2 checks before marking healthy
    "failures_before_critical": 3  // Must fail 3 checks before marking critical
  }
}
```

Your `/health` endpoint should check actual dependencies:

```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    checks := map[string]string{
        "status": "ok",
    }

    // Check database
    if err := db.Ping(); err != nil {
        checks["database"] = "error: " + err.Error()
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(checks)
        return
    }
    checks["database"] = "ok"

    // Check Redis
    if err := redisClient.Ping(ctx).Err(); err != nil {
        checks["redis"] = "degraded: " + err.Error()
        // Don't fail health check for degraded Redis (non-critical)
    } else {
        checks["redis"] = "ok"
    }

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(checks)
}
```

---

## Useful CLI Commands

```bash
# List all registered services
consul catalog services

# List healthy instances of a service
consul health service payments-service --passing

# View all nodes
consul members

# Check cluster status
consul operator raft list-peers

# Watch events in real time
consul monitor

# Force health check run
consul force-leave -prune <node_id>

# Export all KV pairs
consul kv export > kv-backup.json

# Import KV backup
consul kv import @kv-backup.json
```

---

## Related Reading

- [Setting Up pgBouncer for Connection Pooling](/remote-work-tools/setting-up-pgbouncer-for-connection-pooling/)
- [How to Set Up Traefik Reverse Proxy](/remote-work-tools/how-to-set-up-traefik-reverse-proxy/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)

---

## Related Articles

- [Async Product Discovery Process for Remote Teams](/remote-work-tools/async-product-discovery-process-for-remote-teams-using-recorded-interviews/)
- [Setting Up Jaeger for Distributed Tracing](/remote-work-tools/setting-up-jaeger-distributed-tracing/)
- [Remote Agency Client Data Security Compliance Checklist](/remote-work-tools/remote-agency-client-data-security-compliance-checklist-for-proposals/)
- [Setting Up pgBouncer for Connection Pooling](/remote-work-tools/setting-up-pgbouncer-for-connection-pooling/)
- [Setting Up a Remote Dev Server with Hetzner](/remote-work-tools/setting-up-remote-dev-server-with-hetzner/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
