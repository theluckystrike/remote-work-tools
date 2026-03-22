---
layout: default
title: "How to Set Up Thanos for Prometheus HA"
description: "Deploy Thanos Sidecar, Store Gateway, Querier, and Compactor alongside Prometheus for high availability, long-term storage in S3, and unified."
date: 2026-03-22
author: theluckystrike
permalink: /thanos-prometheus-ha-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Set Up Thanos for Prometheus HA

Prometheus stores data locally, which means a single instance has a retention limit based on disk size and goes down with the host it runs on. Thanos wraps Prometheus with object storage (S3, GCS, Azure Blob) for unlimited retention and adds a global query layer that federates across multiple Prometheus instances — useful when remote teams run separate clusters per region.

---

## Architecture Overview

```
┌──────────────────────┐    ┌──────────────────────┐
│  Region: us-east-1   │    │  Region: eu-west-1   │
│                      │    │                      │
│  Prometheus          │    │  Prometheus          │
│    + Thanos Sidecar  │    │    + Thanos Sidecar  │
└──────────┬───────────┘    └──────────┬───────────┘
           │ uploads blocks             │ uploads blocks
           ▼                            ▼
        ┌──────────────────────────────┐
        │         S3 Bucket            │
        └──────────────┬───────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
   ┌──────▼───────┐        ┌────────▼────────┐
   │ Store Gateway│        │   Compactor      │
   │ (reads S3)   │        │ (deduplicates)   │
   └──────┬───────┘        └─────────────────┘
          │
   ┌──────▼───────┐
   │   Querier    │  ← your Grafana queries hit here
   │ (fan out)    │
   └──────────────┘
```

---

## Step 1: Thanos Sidecar on Each Prometheus Instance

The sidecar runs alongside Prometheus, exposes gRPC for the Querier, and uploads TSDB blocks to object storage.

**`docker-compose.yml` on each Prometheus host:**

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.min-block-duration=2h"    # required for Thanos
      - "--storage.tsdb.max-block-duration=2h"    # required for Thanos
      - "--web.enable-lifecycle"
      - "--web.listen-address=0.0.0.0:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  thanos-sidecar:
    image: quay.io/thanos/thanos:v0.35.1
    command:
      - "sidecar"
      - "--tsdb.path=/prometheus"
      - "--prometheus.url=http://prometheus:9090"
      - "--grpc-address=0.0.0.0:10901"
      - "--http-address=0.0.0.0:10902"
      - "--objstore.config-file=/etc/thanos/s3.yml"
    volumes:
      - prometheus-data:/prometheus
      - ./s3.yml:/etc/thanos/s3.yml:ro
    ports:
      - "10901:10901"
      - "10902:10902"
    depends_on:
      - prometheus

volumes:
  prometheus-data:
```

**`s3.yml` — object store config:**

```yaml
type: S3
config:
  bucket: "your-thanos-metrics-bucket"
  endpoint: "s3.amazonaws.com"
  region: "us-east-1"
  access_key: "${AWS_ACCESS_KEY_ID}"
  secret_key: "${AWS_SECRET_ACCESS_KEY}"
  # For MinIO:
  # endpoint: "minio.internal:9000"
  # insecure: true
```

---

## Step 2: Thanos Store Gateway

The Store Gateway exposes historical data from S3 over gRPC so the Querier can access it.

```yaml
# store-gateway docker-compose.yml
version: "3.8"
services:
  thanos-store:
    image: quay.io/thanos/thanos:v0.35.1
    command:
      - "store"
      - "--data-dir=/var/thanos/store"
      - "--objstore.config-file=/etc/thanos/s3.yml"
      - "--grpc-address=0.0.0.0:10901"
      - "--http-address=0.0.0.0:10902"
      - "--sync-block-duration=5m"
    volumes:
      - store-data:/var/thanos/store
      - ./s3.yml:/etc/thanos/s3.yml:ro
    ports:
      - "10901:10901"

volumes:
  store-data:
```

---

## Step 3: Thanos Querier

The Querier fans out PromQL queries to all StoreAPI endpoints (sidecars + store gateway) and deduplicates results from HA replica pairs.

```yaml
# querier docker-compose.yml
version: "3.8"
services:
  thanos-querier:
    image: quay.io/thanos/thanos:v0.35.1
    command:
      - "query"
      - "--http-address=0.0.0.0:10902"
      - "--grpc-address=0.0.0.0:10901"
      # Sidecar endpoints (one per Prometheus instance)
      - "--endpoint=thanos-sidecar-us-east-1.internal:10901"
      - "--endpoint=thanos-sidecar-eu-west-1.internal:10901"
      # Store Gateway for historical data
      - "--endpoint=thanos-store.internal:10901"
      # Deduplicate HA replica pairs by this label
      - "--query.replica-label=replica"
      - "--query.auto-downsampling"
    ports:
      - "9090:10902"  # Grafana points to this
```

Point Grafana at `http://thanos-querier:9090` — it speaks the standard Prometheus HTTP API.

---

## Step 4: Thanos Compactor

The Compactor runs as a singleton (never more than one at a time), downsamples old data, and removes duplicate blocks.

```yaml
# compactor docker-compose.yml
version: "3.8"
services:
  thanos-compactor:
    image: quay.io/thanos/thanos:v0.35.1
    command:
      - "compact"
      - "--data-dir=/var/thanos/compact"
      - "--objstore.config-file=/etc/thanos/s3.yml"
      - "--http-address=0.0.0.0:10902"
      - "--retention.resolution-raw=30d"   # raw data kept 30 days
      - "--retention.resolution-5m=90d"    # 5m resolution kept 90 days
      - "--retention.resolution-1h=1y"     # 1h resolution kept 1 year
      - "--wait"                           # run continuously
      - "--wait-interval=30m"
    volumes:
      - compact-data:/var/thanos/compact
      - ./s3.yml:/etc/thanos/s3.yml:ro

volumes:
  compact-data:
```

---

## Kubernetes Deployment

For production Kubernetes clusters, use the Thanos Helm chart:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# values.yaml
cat > thanos-values.yaml << 'EOF'
objstoreConfig: |
  type: S3
  config:
    bucket: your-thanos-bucket
    endpoint: s3.amazonaws.com
    region: us-east-1

querier:
  enabled: true
  replicaLabel: replica
  stores:
    - thanos-storegateway:10901

storegateway:
  enabled: true
  persistence:
    enabled: true
    size: 20Gi

compactor:
  enabled: true
  persistence:
    enabled: true
    size: 50Gi
  retentionResolutionRaw: 30d
  retentionResolution5m: 90d
  retentionResolution1h: 1y
EOF

helm upgrade --install thanos bitnami/thanos \
  -f thanos-values.yaml \
  -n monitoring \
  --create-namespace
```

---

## Verify the Setup

```bash
# Check store API endpoints visible to Querier
curl http://thanos-querier:9090/api/v1/stores | jq '.'

# Query via the Querier (same as Prometheus API)
curl -G http://thanos-querier:9090/api/v1/query \
  --data-urlencode 'query=up' | jq '.data.result | length'

# Check S3 block uploads (should grow every 2 hours)
aws s3 ls s3://your-thanos-metrics-bucket/ --recursive | head -20

# Verify deduplication is working
curl -G http://thanos-querier:9090/api/v1/query \
  --data-urlencode 'query=count(up)' \
  --data-urlencode 'dedup=true'
```

---

## Troubleshooting Common Thanos Issues

**Sidecar can't upload blocks:** Check that the S3 bucket exists and the access key has `s3:PutObject` and `s3:GetObject` permissions on the bucket. Run `thanos check rules` to validate your config. Look for errors in `docker logs thanos-sidecar`.

**Querier shows no stores:** The Querier lists StoreAPI endpoints via `--endpoint` flags or service discovery. Verify connectivity: `nc -zv thanos-sidecar.internal 10901`. If behind a firewall, open port 10901 for gRPC traffic.

**Duplicate time series in Grafana:** Set `--query.replica-label=replica` on the Querier and ensure each Prometheus instance has `external_labels: {replica: "0"}` or `replica: "1"`. Without the replica label, Thanos can't deduplicate HA pairs.

**Compactor failing with "overlap":** If two Thanos instances ran simultaneously (violates the singleton requirement), S3 may contain overlapping blocks. Run: `thanos tools bucket verify --objstore.config-file s3.yml` to inspect, then `thanos tools bucket replicate` to repair.

**High Store Gateway memory:** By default, the Store Gateway keeps all block indexes in memory. Tune with `--store.grpc.series-max-concurrency` and `--block-sync-concurrency`. For 3+ years of data, consider upgrading to at least 8GB RAM for the store gateway.

**Slow Grafana queries:** Enable downsampling by running the Compactor. Once blocks are compacted at 5m and 1h resolutions, Grafana auto-selects the appropriate resolution based on the time range queried. Without the Compactor, all queries hit raw 15s-resolution data regardless of time range.

```bash
# Debug slow queries — check which stores are queried
curl -G "http://thanos-querier:9090/api/v1/query" \
  --data-urlencode 'query=up' \
  --data-urlencode 'stats=all' | jq '.stats'
```

---

## Securing Thanos for Remote Teams

In a remote team context, the Thanos Querier HTTP endpoint often needs to be accessible to team members in different locations. Do not expose it directly to the internet without authentication.

**Basic auth via Traefik:**
```yaml
labels:
  - "traefik.http.routers.thanos-querier.middlewares=auth"
  - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."
```

**VPN-only access:** The recommended approach for internal observability tooling. Run Thanos on an internal network accessible only via VPN, and expose Grafana (which connects to the Querier internally) via TLS + SSO.

**TLS on gRPC (inter-component):** For production deployments where Thanos components communicate across hosts, add TLS to gRPC:

```yaml
# sidecar command additions
- "--grpc-server-tls-cert=/certs/server.crt"
- "--grpc-server-tls-key=/certs/server.key"
- "--grpc-server-tls-client-ca=/certs/ca.crt"

# querier command additions (matching)
- "--grpc-client-tls-cert=/certs/client.crt"
- "--grpc-client-tls-key=/certs/client.key"
- "--grpc-client-tls-ca=/certs/ca.crt"
```

---

## Cost Optimization

Thanos in S3 storage can accumulate significant costs if retention and downsampling aren't configured correctly.

**Enable retention limits:**
```yaml
# compactor command
- "--retention.resolution-raw=30d"   # raw data: 30 days
- "--retention.resolution-5m=90d"    # 5m downsampled: 3 months
- "--retention.resolution-1h=365d"   # 1h downsampled: 1 year
```

**Use S3 Intelligent-Tiering** for the Thanos bucket — recent blocks are accessed frequently by the store gateway, older blocks rarely. Intelligent-Tiering moves blocks to cheaper storage automatically.

**Estimate storage cost:** A single Prometheus instance generating ~10K active time series produces roughly 1-2GB of TSDB blocks per day. At S3 standard pricing ($0.023/GB), 1 year of retention for one Prometheus instance costs approximately $8-16/month.

---

## Related Reading

- [Prometheus Alerting for Remote Infrastructure](/remote-work-tools/prometheus-alerting-remote-infra-setup/)
- [How to Set Up MinIO for Artifact Storage](/remote-work-tools/minio-artifact-storage-setup/)
- [Prometheus Monitoring Setup for Remote Infrastructure](/remote-work-tools/prometheus-monitoring-remote-infrastructure/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
