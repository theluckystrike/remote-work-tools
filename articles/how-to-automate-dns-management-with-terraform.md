---
layout: default
title: "How to Automate DNS Management with Terraform"
description: "Use Terraform to manage DNS records across Route53, Cloudflare, and other providers with state tracking and team review workflows"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-dns-management-with-terraform/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Manual DNS changes break things and leave no audit trail. Terraform brings DNS under version control with plan/apply workflows that fit remote teams using pull requests. This guide covers Route53 and Cloudflare with shared state, modules, and CI gating.

## Prerequisites

- Terraform 1.6+
- AWS CLI or Cloudflare API token
- S3 bucket for remote state (or Terraform Cloud)

```bash
terraform --version
# Terraform v1.6.x

# Install tfenv for version management
brew install tfenv
tfenv install 1.6.6
tfenv use 1.6.6
```

## Project Structure

```
dns/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
├── versions.tf
├── modules/
│   ├── route53_zone/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── cloudflare_zone/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── staging/
│       ├── main.tf
│       └── terraform.tfvars
└── .github/
    └── workflows/
        └── dns.yml
```

## Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "your-company-terraform-state"
    key            = "dns/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

Create the DynamoDB lock table once:

```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

## Versions and Providers

```hcl
# versions.tf
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

## Route53 Zone Module

```hcl
# modules/route53_zone/variables.tf
variable "domain" {
  description = "Root domain name"
  type        = string
}

variable "records" {
  description = "Map of DNS records"
  type = map(object({
    type    = string
    ttl     = number
    records = list(string)
  }))
  default = {}
}

variable "aliases" {
  description = "Alias records for AWS resources"
  type = map(object({
    name                   = string
    zone_id                = string
    evaluate_target_health = bool
  }))
  default = {}
}
```

```hcl
# modules/route53_zone/main.tf
resource "aws_route53_zone" "this" {
  name = var.domain
}

resource "aws_route53_record" "records" {
  for_each = var.records

  zone_id = aws_route53_zone.this.zone_id
  name    = each.key
  type    = each.value.type
  ttl     = each.value.ttl
  records = each.value.records
}

resource "aws_route53_record" "aliases" {
  for_each = var.aliases

  zone_id = aws_route53_zone.this.zone_id
  name    = each.key
  type    = "A"

  alias {
    name                   = each.value.name
    zone_id                = each.value.zone_id
    evaluate_target_health = each.value.evaluate_target_health
  }
}
```

## Cloudflare Zone Module

```hcl
# modules/cloudflare_zone/main.tf
data "cloudflare_zone" "this" {
  name = var.domain
}

resource "cloudflare_record" "records" {
  for_each = var.records

  zone_id  = data.cloudflare_zone.this.id
  name     = each.key
  type     = each.value.type
  value    = each.value.value
  ttl      = each.value.proxied ? 1 : each.value.ttl
  proxied  = each.value.proxied
  priority = lookup(each.value, "priority", null)

  lifecycle {
    create_before_destroy = true
  }
}

resource "cloudflare_page_rule" "www_redirect" {
  count = var.enable_www_redirect ? 1 : 0

  zone_id  = data.cloudflare_zone.this.id
  target   = "www.${var.domain}/*"
  priority = 1

  actions {
    forwarding_url {
      url         = "https://${var.domain}/$1"
      status_code = 301
    }
  }
}
```

## Production Environment

```hcl
# environments/production/main.tf
module "example_com" {
  source = "../../modules/cloudflare_zone"

  domain              = "example.com"
  cloudflare_api_token = var.cloudflare_api_token
  enable_www_redirect = true

  records = {
    "@" = {
      type    = "A"
      value   = "203.0.113.10"
      ttl     = 1
      proxied = true
    }
    "api" = {
      type    = "A"
      value   = "203.0.113.20"
      ttl     = 1
      proxied = true
    }
    "mail" = {
      type    = "MX"
      value   = "aspmx.l.google.com"
      ttl     = 300
      proxied = false
      priority = 1
    }
    "_dmarc" = {
      type    = "TXT"
      value   = "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
      ttl     = 300
      proxied = false
    }
  }
}
```

## Variables and tfvars

```hcl
# variables.tf
variable "cloudflare_api_token" {
  description = "Cloudflare API token with DNS edit permissions"
  type        = string
  sensitive   = true
}

variable "aws_region" {
  description = "AWS region for Route53"
  type        = string
  default     = "us-east-1"
}
```

```hcl
# environments/production/terraform.tfvars
# Do NOT commit sensitive values - use environment variables or secrets manager
aws_region = "us-east-1"
```

Set secrets via environment:

```bash
export TF_VAR_cloudflare_api_token="your-token-here"
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
```

## Daily Workflow

```bash
# Initialize (first time or after provider changes)
terraform init

# Format all files
terraform fmt -recursive

# Validate configuration
terraform validate

# Plan changes - always review before applying
terraform plan -out=tfplan

# Apply the saved plan
terraform apply tfplan

# Target a specific resource
terraform plan -target=module.example_com.cloudflare_record.records[\"api\"]

# Import existing DNS records (for migrating existing zones)
terraform import 'module.example_com.cloudflare_record.records["api"]' <zone_id>/<record_id>
```

## CI/CD with GitHub Actions

```yaml
# .github/workflows/dns.yml
name: DNS Changes

on:
  pull_request:
    paths: ['dns/**']
  push:
    branches: [main]
    paths: ['dns/**']

env:
  TF_VERSION: "1.6.6"
  WORKING_DIR: "dns/environments/production"

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Plan
        id: plan
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform plan -no-color 2>&1 | tee plan_output.txt
        env:
          TF_VAR_cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}

      - name: Comment PR with plan
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## DNS Plan\n\`\`\`\n${plan.slice(0, 60000)}\n\`\`\``
            });

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Apply
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init && terraform apply -auto-approve
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          TF_VAR_cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

## Drift Detection

Scheduled job to catch manual changes:

```yaml
# Add to dns.yml
  drift-check:
    runs-on: ubuntu-latest
    schedule:
      - cron: '0 8 * * 1'  # Every Monday 8am
    steps:
      - uses: actions/checkout@v4
      - name: Check for drift
        run: terraform plan -detailed-exitcode
        # Exit code 2 = changes detected (drift)
```

## Related Reading

- [Terraform Remote Team Infrastructure Guide](/remote-work-tools/terraform-remote-team-infrastructure-guide/)
- [How to Set Up Ansible for Remote Server Management](/remote-work-tools/how-to-set-up-ansible-remote-server-management/)
- [Best Secrets Management Tool for Remote Dev Teams](/remote-work-tools/best-secrets-management-tool-for-remote-development-teams-us/)
---

## Related Articles

- [Terraform for Remote Teams: State, Modules, and CI](/remote-work-tools/terraform-remote-team-infrastructure-guide/)
- [DNS Filtering Setup for Remote Team Endpoint Security](/remote-work-tools/dns-filtering-setup-for-remote-team-endpoint-security-using-/)
- [AWS Cost Management for Remote Teams](/remote-work-tools/aws-cost-management-remote-teams-guide/)
- [Best Wiki Tool for Remote Team with Version History and](/remote-work-tools/best-wiki-tool-for-remote-team-with-version-history-and-appr/)
- [Migrating from AWS CodeCommit to GitHub for Remote Team](/remote-work-tools/migrating-from-aws-codecommit-to-github-for-remote-team-code/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
