---
layout: default
title: "How to Automate SSL Certificate Renewal"
description: "Automate Let's Encrypt SSL certificate renewal with Certbot, acme.sh, and GitHub Actions for zero-downtime certificate management"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-ssl-certificate-renewal/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Expired SSL certificates take down production services and destroy user trust. Automating renewal removes the human from the loop entirely — certificates rotate before they expire, logs confirm each renewal, and your on-call rotation isn't woken up at 3am because someone forgot to add a calendar reminder.

This guide covers Certbot systemd timers, acme.sh, DNS-01 challenges for wildcard certs, and pipeline-based renewal for non-standard deployments.

---

## Certbot with Systemd Timer (Standard Setup)

When Certbot installs on Debian/Ubuntu via the system package, it creates a systemd timer automatically:

```bash
apt install certbot python3-certbot-nginx

# Issue a certificate
certbot --nginx -d yourcompany.com -d www.yourcompany.com \
  --email ops@yourcompany.com \
  --agree-tos \
  --no-eff-email

# Check the timer
systemctl status certbot.timer
```

Verify the timer is active:

```bash
systemctl list-timers certbot.timer
# Output:
# NEXT                         LEFT          LAST                         PASSED       UNIT
# Sun 2026-03-29 08:34:21 UTC  6 days left   Sun 2026-03-22 08:34:21 UTC  2h ago       certbot.timer
```

The default timer runs twice daily and only renews certificates within 30 days of expiry. That's usually fine. But you also want to validate the renewal works before trusting it:

```bash
# Dry run (no actual renewal)
certbot renew --dry-run

# Force renewal regardless of expiry window (for testing)
certbot renew --force-renewal
```

Add a post-renewal hook to reload your web server:

```bash
# /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
#!/bin/bash
systemctl reload nginx
```

```bash
chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

---

## Certbot DNS-01 for Wildcard Certificates

HTTP-01 challenges don't work for wildcard certs (`*.yourcompany.com`). Use DNS-01 instead.

For Route 53:

```bash
pip install certbot-dns-route53

certbot certonly \
  --dns-route53 \
  -d "*.yourcompany.com" \
  -d yourcompany.com \
  --email ops@yourcompany.com \
  --agree-tos \
  --no-eff-email
```

The Route 53 plugin needs IAM credentials. Create a policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:GetChange"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ChangeResourceRecordSets",
      "Resource": "arn:aws:route53:::hostedzone/YOUR_HOSTED_ZONE_ID"
    }
  ]
}
```

Set credentials via environment or `~/.aws/credentials`:

```bash
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

For Cloudflare DNS:

```bash
pip install certbot-dns-cloudflare

# /etc/letsencrypt/cloudflare.ini
dns_cloudflare_api_token = your_cloudflare_api_token

chmod 600 /etc/letsencrypt/cloudflare.ini

certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d "*.yourcompany.com"
```

---

## acme.sh as a Certbot Alternative

acme.sh is a pure shell script with no Python dependency, useful for embedded systems or minimal containers:

```bash
curl https://get.acme.sh | sh -s email=ops@yourcompany.com

# Issue via DNS-01 with Cloudflare
export CF_Token="your_cloudflare_token"
export CF_Account_ID="your_account_id"

~/.acme.sh/acme.sh --issue \
  --dns dns_cf \
  -d yourcompany.com \
  -d "*.yourcompany.com"

# Install certificate to nginx path
~/.acme.sh/acme.sh --install-cert \
  -d yourcompany.com \
  --cert-file      /etc/nginx/ssl/yourcompany.com.crt \
  --key-file       /etc/nginx/ssl/yourcompany.com.key \
  --fullchain-file /etc/nginx/ssl/yourcompany.com.fullchain.crt \
  --reloadcmd      "systemctl reload nginx"
```

acme.sh installs a cron job automatically:

```bash
crontab -l | grep acme
# 24 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

---

## Monitor Expiry with a Script

Add proactive monitoring so you're alerted before expiry even if renewal fails:

```bash
#!/bin/bash
# /usr/local/bin/check-ssl-expiry.sh
# Sends Slack alert if cert expires within 14 days

DOMAINS=(
  "yourcompany.com"
  "api.yourcompany.com"
  "app.yourcompany.com"
)

SLACK_WEBHOOK="https://hooks.slack.com/services/T.../B.../..."
ALERT_DAYS=14

for domain in "${DOMAINS[@]}"; do
  expiry=$(echo | openssl s_client -servername "$domain" \
    -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null \
    | cut -d= -f2)

  if [ -z "$expiry" ]; then
    msg="SSL CHECK FAILED: Cannot connect to $domain"
    curl -s -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"$msg\"}" "$SLACK_WEBHOOK"
    continue
  fi

  expiry_epoch=$(date -d "$expiry" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$expiry" +%s)
  now_epoch=$(date +%s)
  days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

  if [ "$days_left" -le "$ALERT_DAYS" ]; then
    msg="SSL EXPIRY WARNING: $domain expires in $days_left days ($expiry)"
    curl -s -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"$msg\"}" "$SLACK_WEBHOOK"
  fi
done
```

Add to cron:

```bash
# /etc/cron.d/ssl-expiry-check
0 8 * * * root /usr/local/bin/check-ssl-expiry.sh
```

---

## GitHub Actions for Certificate Distribution

When you issue certs on one server but need them on multiple (CDN edge nodes, load balancers), use GitHub Actions to distribute:

```yaml
# .github/workflows/ssl-renewal.yml
name: SSL Certificate Renewal

on:
  schedule:
    - cron: '0 3 * * 1'  # Every Monday at 3am UTC
  workflow_dispatch:

jobs:
  renew:
    runs-on: ubuntu-latest
    steps:
      - name: Install acme.sh
        run: curl https://get.acme.sh | sh -s email=ops@yourcompany.com

      - name: Renew certificates
        env:
          CF_Token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CF_Account_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        run: |
          ~/.acme.sh/acme.sh --renew \
            --dns dns_cf \
            -d yourcompany.com \
            -d "*.yourcompany.com" \
            --force

      - name: Upload cert to S3
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws s3 cp ~/.acme.sh/yourcompany.com/fullchain.cer \
            s3://your-certs-bucket/yourcompany.com/fullchain.pem \
            --sse aws:kms
          aws s3 cp ~/.acme.sh/yourcompany.com/yourcompany.com.key \
            s3://your-certs-bucket/yourcompany.com/privkey.pem \
            --sse aws:kms

      - name: Notify deployment servers
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" \
            https://deploy.yourcompany.com/api/reload-ssl
```

Each server polls the S3 bucket and reloads Nginx when the cert changes:

```bash
#!/bin/bash
# /usr/local/bin/sync-ssl-cert.sh

BUCKET="s3://your-certs-bucket/yourcompany.com"
LOCAL_PATH="/etc/nginx/ssl"
CERT_CHANGED=false

aws s3 cp "$BUCKET/fullchain.pem" /tmp/fullchain.pem.new
aws s3 cp "$BUCKET/privkey.pem" /tmp/privkey.pem.new

if ! diff -q /tmp/fullchain.pem.new "$LOCAL_PATH/fullchain.pem" > /dev/null 2>&1; then
  cp /tmp/fullchain.pem.new "$LOCAL_PATH/fullchain.pem"
  cp /tmp/privkey.pem.new "$LOCAL_PATH/privkey.pem"
  CERT_CHANGED=true
fi

if $CERT_CHANGED; then
  nginx -t && systemctl reload nginx
  echo "$(date): Certificate updated and Nginx reloaded"
fi
```

---

## Validate Your Renewal Chain Works

Before trusting automation, test the full chain:

```bash
# Check cert details
openssl x509 -in /etc/letsencrypt/live/yourcompany.com/fullchain.pem \
  -noout -text | grep -E "(Subject:|Not After)"

# Test renewal dry run
certbot renew --dry-run --cert-name yourcompany.com

# Verify cert served by Nginx matches the file on disk
echo | openssl s_client -connect yourcompany.com:443 2>/dev/null \
  | openssl x509 -noout -fingerprint

openssl x509 -in /etc/letsencrypt/live/yourcompany.com/cert.pem \
  -noout -fingerprint
```

Both fingerprints should match. If they differ, Nginx is serving a cached or different certificate — reload it:

```bash
nginx -t && systemctl reload nginx
```

---

## Related Reading

- [How to Set Up Traefik Reverse Proxy](/remote-work-tools/how-to-set-up-traefik-reverse-proxy/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Create Automated Status Pages](/remote-work-tools/how-to-create-automated-status-pages/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
