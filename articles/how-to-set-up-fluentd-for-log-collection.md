---
layout: default
title: "How to Set Up Fluentd for Log Collection"
description: "Deploy Fluentd to aggregate logs from Docker containers, applications, and servers into Elasticsearch, S3, or a centralized log store for remote teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-fluentd-for-log-collection/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Fluentd is the CNCF log aggregation standard. It collects logs from dozens of sources (Docker, syslog, application files, Kubernetes), transforms and filters them, then routes to one or more destinations. For remote teams with multiple services across multiple hosts, centralized logging is the difference between having observable systems and debugging in the dark.

---

## Install Fluentd

On Debian/Ubuntu (using td-agent, the stable distribution):

```bash
curl -fsSL https://toolbelt.treasuredata.com/sh/install-ubuntu-focal-td-agent4.sh | sh
systemctl enable td-agent
systemctl start td-agent
```

Docker (recommended for containerized infrastructure):

```bash
docker pull fluent/fluentd:v1.17-1
```

---

## Basic Configuration Structure

Fluentd config lives at `/etc/td-agent/td-agent.conf`. It has four directive types:

- `<source>`: Where to read logs from
- `<filter>`: Transform and enrich records
- `<match>`: Where to send logs
- `<label>`: Group directives for routing

A minimal config that reads Docker logs and sends to Elasticsearch:

```xml
# /etc/td-agent/td-agent.conf

# =====================
# SOURCES
# =====================

# Collect Docker container logs
<source>
  @type tail
  @id docker_logs
  path /var/lib/docker/containers/*/*.log
  pos_file /var/log/td-agent/docker.log.pos
  tag docker.*
  read_from_head true
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

# Collect syslog
<source>
  @type syslog
  @id syslog
  port 5140
  bind 0.0.0.0
  tag system
  <transport udp>
  </transport>
</source>

# Collect application logs via HTTP API
<source>
  @type http
  @id http_input
  port 8888
  bind 0.0.0.0
  body_size_limit 32m
  keepalive_timeout 10
</source>

# =====================
# FILTERS
# =====================

# Parse Docker log metadata
<filter docker.**>
  @type record_transformer
  <record>
    container_id ${tag_suffix[4]}
    host "#{Socket.gethostname}"
    environment "#{ENV['ENVIRONMENT'] || 'production'}"
  </record>
</filter>

# Drop health check noise
<filter docker.**>
  @type grep
  <exclude>
    key log
    pattern /GET \/health/
  </exclude>
</filter>

# Parse JSON application logs
<filter docker.**>
  @type parser
  key_name log
  reserve_data true
  <parse>
    @type json
    json_parser json
  </parse>
</filter>

# =====================
# OUTPUT
# =====================

<match docker.**>
  @type elasticsearch
  @id elasticsearch_out
  host elasticsearch.yourcompany.internal
  port 9200
  user elastic
  password "#{ENV['ES_PASSWORD']}"
  logstash_format true
  logstash_prefix docker
  logstash_dateformat %Y.%m.%d
  flush_interval 10s
  <buffer>
    @type file
    path /var/log/td-agent/buffer/elasticsearch
    flush_mode interval
    flush_interval 10s
    chunk_limit_size 8m
    queue_limit_length 128
    retry_max_times 10
    retry_wait 30s
    retry_exponential_backoff_base 2
    overflow_action block
  </buffer>
</match>
```

---

## Docker Compose Deployment

For a full logging stack with Docker:

```yaml
# docker-compose.yml
version: "3.8"

services:
  fluentd:
    build:
      context: ./fluentd
      dockerfile: Dockerfile
    container_name: fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - fluentd-pos:/fluentd/pos
      - fluentd-buffer:/fluentd/buffer
    ports:
      - "24224:24224"     # Fluentd forward protocol
      - "24224:24224/udp"
      - "8888:8888"       # HTTP input
    environment:
      - ES_PASSWORD=${ES_PASSWORD}
      - ENVIRONMENT=production
    restart: unless-stopped
    networks:
      - logging

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - logging

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - logging

networks:
  logging:

volumes:
  es-data:
  fluentd-pos:
  fluentd-buffer:
```

Fluentd Dockerfile with required plugins:

```dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v1.17-1

USER root

# Install plugins
RUN gem install \
    fluent-plugin-elasticsearch \
    fluent-plugin-s3 \
    fluent-plugin-rewrite-tag-filter \
    fluent-plugin-record-modifier

USER fluent
```

---

## Configure Applications to Send Logs to Fluentd

Applications send logs to the Fluentd container via the forward protocol:

```yaml
# In your app's docker-compose.yml
services:
  api:
    image: yourcompany/api:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: "fluentd:24224"
        fluentd-async: "true"
        tag: "{{.Name}}"
```

From application code (Go):

```go
import (
    "github.com/fluent/fluent-logger-golang/fluent"
)

func main() {
    logger, err := fluent.New(fluent.Config{
        FluentHost: "fluentd",
        FluentPort: 24224,
        TagPrefix:  "app",
    })
    if err != nil {
        panic(err)
    }
    defer logger.Close()

    // Log structured event
    logger.Post("payments", map[string]interface{}{
        "user_id":    userID,
        "amount":     amount,
        "currency":   currency,
        "request_id": requestID,
        "level":      "info",
        "message":    "payment processed",
    })
}
```

---

## Route Logs to S3 for Long-Term Storage

```xml
# Add to td-agent.conf

<match **>
  @type s3
  @id s3_archive
  aws_key_id "#{ENV['AWS_ACCESS_KEY_ID']}"
  aws_sec_key "#{ENV['AWS_SECRET_ACCESS_KEY']}"
  s3_bucket your-log-archive-bucket
  s3_region us-east-1
  path logs/%Y/%m/%d/
  s3_object_key_format %{path}%{time_slice}_%{index}.%{file_extension}
  time_slice_format %Y%m%d_%H
  time_slice_wait 10m
  store_as gzip
  <format>
    @type json
  </format>
  <buffer time>
    @type file
    path /fluentd/buffer/s3
    timekey 3600          # 1 hour chunks
    timekey_wait 10m      # Wait 10 minutes before flushing to allow late records
    chunk_limit_size 256m
  </buffer>
</match>
```

---

## Multi-Output Routing with Labels

Send logs to different destinations based on content:

```xml
<match docker.**>
  @type relabel
  @label @routing
</match>

<label @routing>
  # Security events → dedicated security log store
  <match **>
    @type rewrite_tag_filter
    <rule>
      key level
      pattern /SECURITY|AUDIT/
      tag security.${tag}
    </rule>
    <rule>
      key $.http_status
      pattern /^5/
      tag errors.${tag}
    </rule>
  </match>

  <match security.**>
    @type elasticsearch
    host security-elasticsearch.yourcompany.internal
    port 9200
    logstash_prefix security
    logstash_format true
  </match>

  <match errors.**>
    @type copy
    <store>
      @type elasticsearch
      host elasticsearch.yourcompany.internal
      port 9200
      logstash_prefix errors
    </store>
    <store>
      @type slack
      webhook_url "#{ENV['SLACK_WEBHOOK']}"
      channel "#alerts"
      message_keys level,message,container_id
      title "Application Error"
    </store>
  </match>

  <match **>
    @type elasticsearch
    host elasticsearch.yourcompany.internal
    port 9200
    logstash_prefix app
    logstash_format true
  </match>
</label>
```

---

## Monitor Fluentd Health

```bash
# Check Fluentd status
curl http://localhost:24220/api/config.json | jq '.'

# Metrics endpoint
curl http://localhost:24220/api/plugins.json | jq '.'

# Check buffer queue depth (high = Elasticsearch can't keep up)
curl http://localhost:24220/api/plugins.json \
  | jq '.plugins[] | select(.plugin_id | contains("elasticsearch")) | {buffer_queue_length, retry_count}'
```

---

## Related Reading

- [How to Set Up Vector for Log Processing](/remote-work-tools/how-to-set-up-vector-for-log-processing/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [Best Observability Platform for Remote Teams Correlating](/remote-work-tools/best-observability-platform-for-remote-teams-correlating-log/)

---

## Related Articles

- [How to Set Up Vector for Log Processing](/remote-work-tools/how-to-set-up-vector-for-log-processing/)
- [Optimize Docker for Slow Connections When Working Remotely](/remote-work-tools/docker-optimize-slow-connection-remote-work/)
- [Nix vs Docker for Reproducible Dev Environments](/remote-work-tools/nix-vs-docker-for-reproducible-dev-environments/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
- [How to Create Decision Log Documentation for Remote Teams](/remote-work-tools/how-to-create-decision-log-documentation-for-remote-teams-re/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
