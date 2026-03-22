---
layout: default
title: "Setting Up Keycloak for Team SSO"
description: "Deploy Keycloak for self-hosted team SSO with OIDC, configure app clients, sync with LDAP or Google, and enforce MFA for remote teams"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-keycloak-for-team-sso/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Keycloak gives remote teams a self-hosted identity provider. One login for Gitea, Grafana, BookStack, SonarQube, and any other internal tool. This guide covers Docker deployment, realm setup, OIDC app clients, Google federation, and MFA enforcement.

## Docker Compose Deployment

```yaml
# docker-compose.yml
version: "3.8"

services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.4
    container_name: keycloak
    command: start --optimized
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak_db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${DB_PASSWORD}
      KC_HOSTNAME: auth.example.com
      KC_HOSTNAME_STRICT: "true"
      KC_PROXY: edge
      KC_HTTP_ENABLED: "true"
      KC_HEALTH_ENABLED: "true"
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
    ports:
      - "8080:8080"
    depends_on:
      - keycloak_db
    restart: unless-stopped

  keycloak_db:
    image: postgres:15-alpine
    container_name: keycloak_db
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - keycloak_db_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  keycloak_db_data:
```

```bash
# .env
DB_PASSWORD=strong-db-password
ADMIN_PASSWORD=strong-admin-password
```

```bash
docker compose up -d
docker compose logs -f keycloak
# Wait for: Admin console listening on http://0.0.0.0:8080/auth
```

## Nginx Reverse Proxy

```nginx
server {
    listen 443 ssl http2;
    server_name auth.example.com;

    ssl_certificate /etc/letsencrypt/live/auth.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/auth.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
}
```

## Realm Configuration

```bash
# Access admin console: https://auth.example.com/admin
# Login with admin credentials

# Create realm via CLI (kcadm.sh)
docker exec keycloak /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user admin \
  --password "$ADMIN_PASSWORD"

# Create company realm
docker exec keycloak /opt/keycloak/bin/kcadm.sh create realms \
  -s realm=company \
  -s enabled=true \
  -s displayName="Company SSO" \
  -s registrationAllowed=false \
  -s resetPasswordAllowed=true \
  -s rememberMe=true \
  -s bruteForceProtected=true \
  -s permanentLockout=false \
  -s maxFailureWaitSeconds=900 \
  -s minimumQuickLoginWaitSeconds=60 \
  -s waitIncrementSeconds=60 \
  -s quickLoginCheckMilliSeconds=1000 \
  -s maxDeltaTimeSeconds=43200 \
  -s failureFactor=30
```

## OIDC Client for Grafana

```bash
# Create OIDC client for Grafana
docker exec keycloak /opt/keycloak/bin/kcadm.sh create clients \
  -r company \
  -s clientId=grafana \
  -s name="Grafana" \
  -s enabled=true \
  -s protocol=openid-connect \
  -s publicClient=false \
  -s standardFlowEnabled=true \
  -s directAccessGrantsEnabled=false \
  -s serviceAccountsEnabled=false \
  -s "redirectUris=[\"https://grafana.example.com/login/generic_oauth\"]" \
  -s "webOrigins=[\"https://grafana.example.com\"]"

# Get client secret
docker exec keycloak /opt/keycloak/bin/kcadm.sh get clients \
  -r company \
  --fields id,clientId,secret \
  -q clientId=grafana
```

```ini
# grafana.ini - configure OIDC
[auth.generic_oauth]
enabled = true
name = Company SSO
allow_sign_up = true
client_id = grafana
client_secret = your-client-secret
scopes = openid email profile
auth_url = https://auth.example.com/realms/company/protocol/openid-connect/auth
token_url = https://auth.example.com/realms/company/protocol/openid-connect/token
api_url = https://auth.example.com/realms/company/protocol/openid-connect/userinfo
role_attribute_path = contains(groups[*], 'grafana-admins') && 'Admin' || contains(groups[*], 'grafana-editors') && 'Editor' || 'Viewer'
```

## OIDC Client for Gitea

```bash
# Create Gitea OIDC client
docker exec keycloak /opt/keycloak/bin/kcadm.sh create clients \
  -r company \
  -s clientId=gitea \
  -s enabled=true \
  -s protocol=openid-connect \
  -s publicClient=false \
  -s "redirectUris=[\"https://git.example.com/user/oauth2/keycloak/callback\"]"
```

In Gitea admin (Site Administration > Authentication > Add Authentication Source):
```
Authentication Type: OAuth2
Name: Keycloak
OAuth2 Provider: OpenID Connect
Client ID: gitea
Client Secret: your-secret
OpenID Connect Auto Discovery URL: https://auth.example.com/realms/company/.well-known/openid-configuration
```

## Google Identity Federation

Let team members log in with their Google workspace accounts:

```bash
# In Keycloak admin console:
# Company realm > Identity Providers > Add provider > Google

# Or via API:
docker exec keycloak /opt/keycloak/bin/kcadm.sh create identity-provider/instances \
  -r company \
  -s alias=google \
  -s providerId=google \
  -s enabled=true \
  -s 'config.clientId=your-google-client-id.apps.googleusercontent.com' \
  -s 'config.clientSecret=your-google-client-secret' \
  -s 'config.hostedDomain=example.com' \
  -s 'config.syncMode=IMPORT'
```

## MFA Enforcement

```bash
# Create authentication flow requiring OTP
# In admin console: Authentication > Flows > Create flow

# Or set realm default to require OTP:
docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/company \
  -s 'otpPolicyType=totp' \
  -s 'otpPolicyAlgorithm=HmacSHA1' \
  -s 'otpPolicyInitialCounter=0' \
  -s 'otpPolicyDigits=6' \
  -s 'otpPolicyLookAheadWindow=1' \
  -s 'otpPolicyPeriod=30'

# Require OTP for all users in realm:
# Authentication > Required Actions > Configure OTP > Default Action = ON
```

## User and Group Management

```bash
# Create groups
docker exec keycloak /opt/keycloak/bin/kcadm.sh create groups \
  -r company -s name=engineers
docker exec keycloak /opt/keycloak/bin/kcadm.sh create groups \
  -r company -s name=admins

# Create user
docker exec keycloak /opt/keycloak/bin/kcadm.sh create users \
  -r company \
  -s username=alice \
  -s email=alice@example.com \
  -s firstName=Alice \
  -s lastName=Smith \
  -s enabled=true \
  -s emailVerified=true

# Set password
docker exec keycloak /opt/keycloak/bin/kcadm.sh set-password \
  -r company \
  --username alice \
  --new-password TempPassword123!

# Add to group
ALICE_ID=$(docker exec keycloak /opt/keycloak/bin/kcadm.sh get users -r company -q username=alice --fields id | jq -r '.[0].id')
ENGINEERS_ID=$(docker exec keycloak /opt/keycloak/bin/kcadm.sh get groups -r company -q search=engineers --fields id | jq -r '.[0].id')
docker exec keycloak /opt/keycloak/bin/kcadm.sh update users/$ALICE_ID/groups/$ENGINEERS_ID -r company -s realm=company -s userId=$ALICE_ID -s groupId=$ENGINEERS_ID -n
```

## Backup

```bash
#!/bin/bash
# scripts/backup-keycloak.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/keycloak"
mkdir -p "$BACKUP_DIR"

# Export realm configuration
docker exec keycloak /opt/keycloak/bin/kc.sh export \
  --dir /tmp/keycloak-export \
  --realm company

docker cp keycloak:/tmp/keycloak-export "$BACKUP_DIR/realm-${DATE}"

# Backup database
docker exec keycloak_db pg_dump \
  -U keycloak keycloak | gzip > "$BACKUP_DIR/db-${DATE}.sql.gz"

echo "Keycloak backup complete: $BACKUP_DIR"
```

## Token Session Tuning

Default Keycloak session settings are generous. For a remote team where security matters, tighten them:

```bash
# Shorten access token lifetime (default 5 minutes is fine; refresh token is the session)
docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/company \
  -s accessTokenLifespan=300 \
  -s ssoSessionIdleTimeout=3600 \
  -s ssoSessionMaxLifespan=36000 \
  -s offlineSessionIdleTimeout=2592000

# Force re-authentication after idle (important for unattended shared machines)
# ssoSessionIdleTimeout=3600 means: log out after 1 hour of inactivity
# ssoSessionMaxLifespan=36000 means: force full re-login after 10 hours regardless
```

For tools like Grafana or Gitea that embed long-lived tokens in cookies, make sure your OIDC client has `accessType=CONFIDENTIAL` and that the application refresh-token logic is enabled. Otherwise users will get silent 401 errors when the short-lived access token expires.

## Role-Based Access with Client Roles

Map Keycloak roles to application-level permissions:

```bash
# Create a client role on the Grafana client
CLIENT_ID=$(docker exec keycloak /opt/keycloak/bin/kcadm.sh get clients \
  -r company -q clientId=grafana --fields id | jq -r '.[0].id')

docker exec keycloak /opt/keycloak/bin/kcadm.sh create \
  clients/$CLIENT_ID/roles \
  -r company \
  -s name=grafana-admins \
  -s description="Grafana administrator access"

docker exec keycloak /opt/keycloak/bin/kcadm.sh create \
  clients/$CLIENT_ID/roles \
  -r company \
  -s name=grafana-editors \
  -s description="Grafana editor access"

# Assign role to a user
USER_ID=$(docker exec keycloak /opt/keycloak/bin/kcadm.sh get users \
  -r company -q username=alice --fields id | jq -r '.[0].id')

docker exec keycloak /opt/keycloak/bin/kcadm.sh add-roles \
  -r company \
  --uid $USER_ID \
  --cclientid grafana \
  --rolename grafana-editors
```

Then in `grafana.ini`, the `role_attribute_path` JMESPath expression maps the token's `resource_access.grafana.roles` array to Grafana role strings — this is already shown in the OIDC client section above.

## Keycloak Events and Audit Logging

Track who logged in, what failed, and when tokens were issued:

```bash
# Enable event logging on the realm
docker exec keycloak /opt/keycloak/bin/kcadm.sh update realms/company \
  -s eventsEnabled=true \
  -s eventsExpiration=2592000 \
  -s 'enabledEventTypes=["LOGIN","LOGIN_ERROR","LOGOUT","REGISTER","REGISTER_ERROR",
    "CODE_TO_TOKEN","CLIENT_LOGIN","CLIENT_LOGIN_ERROR","TOKEN_EXCHANGE",
    "REFRESH_TOKEN","REFRESH_TOKEN_ERROR"]' \
  -s adminEventsEnabled=true \
  -s adminEventsDetailsEnabled=true

# Query recent login events via API
curl -s "https://auth.example.com/admin/realms/company/events?type=LOGIN_ERROR&max=50" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[] | {user: .userId, ip: .ipAddress, time: .time}'
```

Forward events to a SIEM or logging platform by configuring the Keycloak Event Listener SPI. The built-in `jboss-logging` listener writes to container stdout, which your log aggregator (Loki, CloudWatch, Datadog) can pick up automatically. For structured output, add the `event-listener-email` or a custom HTTP listener via the admin console under Realm Settings > Events > Event Listeners.

## Upgrading Keycloak

Keycloak's `start --optimized` mode requires rebuilding the image when you add providers or change build-time options. For version upgrades:

```bash
# 1. Backup first (always)
./scripts/backup-keycloak.sh

# 2. Update image tag in docker-compose.yml
# image: quay.io/keycloak/keycloak:24.0.1

# 3. Pull and restart — Keycloak runs DB migrations automatically
docker compose pull keycloak
docker compose up -d keycloak

# 4. Watch logs for migration completion
docker compose logs -f keycloak | grep -E "migration|started|error"

# 5. Verify admin console is accessible
curl -s -o /dev/null -w "%{http_code}" https://auth.example.com/admin/
# Should return 200
```

Between major versions (e.g., 22 → 23 → 24), review the Keycloak migration guide. Breaking changes are rare but do affect custom themes and deprecated grant types. Running a staging Keycloak instance that mirrors production is strongly recommended for teams with more than 10 connected applications.

## Related Reading

- [Best Password Manager for a Remote Startup of 15 Employees](/remote-work-tools/best-password-manager-for-a-remote-startup-of-15-employees/)
- [Best Endpoint Security for Remote Employees](/remote-work-tools/best-endpoint-security-solution-for-remote-employees-using-p/)
- [How to Set Up Gitea for Self-Hosted Git](/remote-work-tools/how-to-set-up-gitea-self-hosted-git/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
