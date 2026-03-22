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

---

## Single-Node Install with Docker

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

---

## Multi-Node Setup with Docker Compose

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

---

## Configure with the MinIO Client (mc)

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

---

## Bucket Policies and Access Control

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

---

## Using MinIO from CI/CD as an S3 Drop-In

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

---

## Lifecycle Rules for Automatic Cleanup

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

---

## TLS with Let's Encrypt

```bash
mkdir -p /data/minio/.minio/certs
cp /etc/letsencrypt/live/minio.yourcompany.com/fullchain.pem \
   /data/minio/.minio/certs/public.crt
cp /etc/letsencrypt/live/minio.yourcompany.com/privkey.pem \
   /data/minio/.minio/certs/private.key
chown -R minio:minio /data/minio/.minio/certs

docker restart minio
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

Key alerts: `minio_cluster_capacity_usable_free_bytes < 10GB`, `minio_s3_requests_errors_total`.

---

## Related Reading

- [How to Set Up Thanos for Prometheus HA](/remote-work-tools/thanos-prometheus-ha-setup/)
- [Best Tools for Remote Team Wiki Maintenance](/remote-work-tools/remote-team-wiki-maintenance-tools/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
