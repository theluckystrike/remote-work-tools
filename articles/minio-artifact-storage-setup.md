---
layout: default
title: "How to Set Up MinIO for Artifact Storage"
description: "Deploy MinIO as a self-hosted S3-compatible artifact store for CI/CD build outputs, Terraform state, and ML datasets with lifecycle rules, RBAC, and TLS"
date: 2026-03-22
author: theluckystrike
permalink: /minio-artifact-storage-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Set Up MinIO for Artifact Storage

Every CI/CD pipeline produces artifacts: binaries, test reports, Docker layers, Terraform plans, ML model checkpoints. Pushing these to S3 adds latency and egress costs. MinIO gives you an S3-compatible object store that runs on your own hardware, uses the same AWS SDK calls, and costs nothing per-request.

This guide covers a full production MinIO setup: single-node for getting started, multi-node for resilience, access control policies, CI/CD integration, lifecycle rules, TLS, monitoring, and common troubleshooting.

---

## Why MinIO Instead of S3

The case for self-hosted artifact storage is straightforward for teams running their own infrastructure. S3 egress can run $0.09/GB, which adds up fast when CI jobs pull multi-gigabyte Docker layers or ML datasets repeatedly. MinIO's egress cost is zero — it's your hardware.

Beyond cost, MinIO solves a few other problems:

- **Latency**: Artifacts stored in the same data center or VPC as your build runners are retrieved in milliseconds, not hundreds of milliseconds. Fast artifact retrieval keeps CI jobs tight.
- **Air-gapped environments**: Regulated industries and government contractors often cannot push artifacts to public cloud. MinIO runs entirely on-prem.
- **S3-compatible API**: Every tool that talks to S3 — Terraform, the AWS CLI, boto3, Rclone, Restic — talks to MinIO without code changes. You only change the endpoint URL.
- **Unified storage**: One MinIO cluster can hold CI artifacts, Terraform state, ML datasets, database backups, and application uploads. Fewer systems to operate.

MinIO is not the right choice if you need a managed service with zero operations overhead. If your team has the runway to manage a storage service, the economics favor MinIO at any meaningful scale.

---

## Single-Node Install with Docker

For a single-developer setup or a small team, one node is enough. Use Docker for the simplest deployment path:

```bash
mkdir -p /data/minio

docker run -d \
  --name minio \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD="$(openssl rand -base64 32)" \
  -e MINIO_VOLUMES="/data" \
  -v /data/minio:/data \
  quay.io/minio/minio server /data --console-address ":9001"
```

Browse the console at `http://your-host:9001`. The API is on port 9000.

Save your generated password immediately — it only appears at container creation time. Store it in your team's secret manager, not in a shell history file.

To retrieve the running container's environment for verification:

```bash
docker inspect minio | grep -A2 MINIO_ROOT
```

---

## Multi-Node Setup with Docker Compose

Four-node erasure-coded deployment tolerates the loss of two nodes without data loss. MinIO requires an even number of nodes (or drives) for erasure coding — four is the minimum recommended for production:

```yaml
version: "3.8"

x-minio-common: &minio-common
  image: quay.io/minio/minio:latest
  command: server http://minio{1...4}/data --console-address ":9001"
  environment:
    MINIO_ROOT_USER: ${MINIO_ROOT_USER}
    MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
  restart: unless-stopped

services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - /mnt/disk1/data:/data

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - /mnt/disk2/data:/data

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - /mnt/disk3/data:/data

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - /mnt/disk4/data:/data
```

Each node should be on a separate physical disk. The erasure coding overhead is 50% — 10 TB of raw disk yields about 5 TB of usable space in a four-node setup. For larger clusters, 8 or 16 nodes reduce that overhead ratio.

Deploy and verify:

```bash
docker compose up -d
docker compose logs -f --tail=50 minio1
# Look for: "MinIO Object Storage Server" and "Console:" lines
```

---

## Configure with the MinIO Client (mc)

The `mc` CLI is the primary management tool. Install it once and alias your cluster:

```bash
curl -LO https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

mc alias set artifacts http://minio.internal:9000 minioadmin "your-password"

mc mb artifacts/ci-build-outputs
mc mb artifacts/terraform-state
mc mb artifacts/ml-datasets
mc mb artifacts/docker-cache

mc ls artifacts/
```

Useful day-to-day commands:

```bash
# Check cluster health
mc admin info artifacts

# View disk usage per bucket
mc du artifacts/ci-build-outputs

# Copy objects between buckets
mc cp artifacts/ci-build-outputs/v1.2.0/ artifacts/ci-build-outputs-archive/v1.2.0/ --recursive

# Mirror to another MinIO cluster (disaster recovery)
mc mirror artifacts/terraform-state artifacts-dr/terraform-state
```

---

## Bucket Policies and Access Control

Service accounts with minimal permissions are safer than sharing root credentials with CI runners. Create a dedicated policy for each use case:

```bash
cat > ci-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::ci-build-outputs",
        "arn:aws:s3:::ci-build-outputs/*"
      ]
    }
  ]
}
EOF

mc admin policy create artifacts ci-write ci-policy.json
mc admin user add artifacts ci-runner "$(openssl rand -base64 24)"
mc admin policy attach artifacts ci-write --user ci-runner
mc admin user svcacct add artifacts ci-runner
```

For read-only access to ML datasets for training jobs:

```bash
cat > ml-read-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::ml-datasets",
        "arn:aws:s3:::ml-datasets/*"
      ]
    }
  ]
}
EOF

mc admin policy create artifacts ml-read ml-read-policy.json
mc admin user add artifacts ml-trainer "$(openssl rand -base64 24)"
mc admin policy attach artifacts ml-read --user ml-trainer
```

Group-based RBAC works for larger teams:

```bash
mc admin group add artifacts ci-team ci-runner1 ci-runner2 ci-runner3
mc admin policy attach artifacts ci-write --group ci-team
```

---

## Using MinIO from CI/CD as an S3 Drop-In

MinIO is fully compatible with the AWS SDK. The only change is setting `endpoint_url` (or the equivalent environment variable).

**GitHub Actions:**

```yaml
- name: Upload build artifact to MinIO
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.MINIO_ACCESS_KEY }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.MINIO_SECRET_KEY }}
    AWS_ENDPOINT_URL: https://minio.internal:9000
    AWS_DEFAULT_REGION: us-east-1
  run: |
    aws s3 cp ./dist/myapp.tar.gz \
      s3://ci-build-outputs/${{ github.sha }}/myapp.tar.gz \
      --endpoint-url $AWS_ENDPOINT_URL
```

**GitLab CI:**

```yaml
upload-artifacts:
  image: amazon/aws-cli
  script:
    - aws s3 sync ./dist/ s3://ci-build-outputs/$CI_COMMIT_SHA/
      --endpoint-url $MINIO_ENDPOINT
  variables:
    AWS_ACCESS_KEY_ID: $MINIO_ACCESS_KEY
    AWS_SECRET_ACCESS_KEY: $MINIO_SECRET_KEY
    AWS_DEFAULT_REGION: us-east-1
```

**Python upload with presigned URL:**

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="https://minio.internal:9000",
    aws_access_key_id="your-access-key",
    aws_secret_access_key="your-secret-key",
    region_name="us-east-1",
)

s3.upload_file(
    "test-results.xml",
    "ci-build-outputs",
    f"tests/{commit_sha}/results.xml",
)

url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "ci-build-outputs", "Key": f"tests/{commit_sha}/results.xml"},
    ExpiresIn=3600,
)
print(f"Test report: {url}")
```

Presigned URLs are useful for sharing test reports or build artifacts with people who do not have MinIO credentials. The URL expires after the specified number of seconds and requires no authentication to access.

---

## Terraform State Backend

Storing Terraform state in MinIO avoids the need for an S3 bucket with state locking. Use DynamoDB-equivalent locking via a separate backend or enable state locking with the HTTP backend:

```hcl
terraform {
  backend "s3" {
    bucket                      = "terraform-state"
    key                         = "prod/network/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "https://minio.internal:9000"
    access_key                  = "your-access-key"
    secret_key                  = "your-secret-key"
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}
```

The `force_path_style = true` setting is required for MinIO. AWS S3 uses virtual-hosted-style URLs by default; MinIO uses path-style.

---

## Lifecycle Rules for Automatic Cleanup

CI artifacts accumulate fast. A branch that builds 10 times per day produces 300 artifact sets per month. Without cleanup, the bucket grows without bound:

```bash
mc ilm rule add \
  --expiry-days 30 \
  artifacts/ci-build-outputs

mc ilm rule add \
  --expired-object-delete-marker \
  --noncurrent-expire-days 3 \
  artifacts/ci-build-outputs

mc ilm rule ls artifacts/ci-build-outputs
```

For ML datasets, tag objects by version and expire old versions:

```bash
mc ilm rule add \
  --expiry-days 90 \
  --tags "environment=experiment" \
  artifacts/ml-datasets
```

Lifecycle rules run on MinIO's internal scheduler. Check when rules last executed:

```bash
mc admin scanner status artifacts
```

---

## TLS with Let's Encrypt

MinIO should always run with TLS in production. Using Let's Encrypt certificates:

```bash
mkdir -p /data/minio/.minio/certs
cp /etc/letsencrypt/live/minio.yourcompany.com/fullchain.pem \
   /data/minio/.minio/certs/public.crt
cp /etc/letsencrypt/live/minio.yourcompany.com/privkey.pem \
   /data/minio/.minio/certs/private.key
chown -R minio:minio /data/minio/.minio/certs

docker restart minio
```

For automatic certificate renewal, add a renewal hook:

```bash
cat > /etc/letsencrypt/renewal-hooks/deploy/minio.sh << 'EOF'
#!/bin/bash
cp /etc/letsencrypt/live/minio.yourcompany.com/fullchain.pem \
   /data/minio/.minio/certs/public.crt
cp /etc/letsencrypt/live/minio.yourcompany.com/privkey.pem \
   /data/minio/.minio/certs/private.key
docker restart minio
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/minio.sh
```

---

## Monitoring MinIO

MinIO exposes Prometheus metrics at `/minio/v2/metrics/cluster`:

```yaml
scrape_configs:
  - job_name: minio
    metrics_path: /minio/v2/metrics/cluster
    scheme: https
    static_configs:
      - targets: ["minio.internal:9000"]
    bearer_token: "your-prometheus-token"
```

Key alerts to set up:

```yaml
groups:
  - name: minio
    rules:
      - alert: MinIOLowDiskSpace
        expr: minio_cluster_capacity_usable_free_bytes < 10737418240
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO usable disk space below 10GB"

      - alert: MinIOHighErrorRate
        expr: rate(minio_s3_requests_errors_total[5m]) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MinIO request error rate elevated"
```

The MinIO console at port 9001 also provides a real-time dashboard covering request rates, bandwidth, and capacity utilization without any additional setup.

---

## Common Issues and Fixes

**"Signature mismatch" errors from AWS SDK**: Check that your system clock is synchronized. S3 request signing is time-sensitive. Run `chronyc tracking` or `timedatectl` to verify NTP sync.

**Objects not replicating in multi-node setup**: Ensure all nodes can resolve each other's hostnames. In Docker Compose, the hostnames are the service names. Add entries to `/etc/hosts` if needed for bare-metal setups.

**Bucket versioning and delete markers accumulating**: Enable the `--expired-object-delete-marker` lifecycle rule alongside your expiry rule to clean up delete markers left by versioned object expiration.

**mc alias shows "connection refused"**: Verify the port is not blocked by a firewall rule. Check `ufw status` or `iptables -L` on the host.

---

## Related Reading

- [How to Set Up Thanos for Prometheus HA](/remote-work-tools/thanos-prometheus-ha-setup/)
- [Best Tools for Remote Team Wiki Maintenance](/remote-work-tools/remote-team-wiki-maintenance-tools/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
