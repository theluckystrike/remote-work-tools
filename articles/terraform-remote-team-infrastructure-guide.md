---
layout: default
title: "Terraform for Remote Teams: State, Modules, and CI"
description: "Set up Terraform for distributed remote teams with remote state, reusable modules, workspace separation, and CI/CD integration. Includes practical config"
date: 2026-03-21
author: theluckystrike
permalink: /terraform-remote-team-infrastructure-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

Running Terraform on a team requires solving problems that solo use does not face: multiple people applying changes simultaneously, shared state that needs locking, and environments (dev, staging, prod) that should be isolated. Without proper setup, remote teams corrupt state files, conflict on applies, and drift environments apart.

This guide covers the complete Terraform setup for remote teams: S3 backend with DynamoDB locking, workspace separation, reusable modules, and GitHub Actions integration.

## Remote State with S3 and DynamoDB Locking

Local `terraform.tfstate` files break immediately in a team. Two people cannot run `terraform apply` simultaneously without corrupting state. S3 + DynamoDB gives you shared state with pessimistic locking.

```bash
# Create the S3 bucket and DynamoDB table for remote state
# Run once per AWS account — before creating any other infrastructure

aws s3api create-bucket \
  --bucket mycompany-terraform-state \
  --region us-east-1

# Enable versioning (recover from state corruption)
aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket mycompany-terraform-state \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket mycompany-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

## Backend Configuration

```hcl
# backend.tf — add to every Terraform project
terraform {
  required_version = ">= 1.7.0"

  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "myapp/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Each project uses a unique `key` path. Convention: `project/environment/terraform.tfstate`.

## Workspace Strategy for Environment Separation

```bash
# Create workspaces for each environment
terraform workspace new dev
terraform workspace new staging
terraform workspace new production

# List workspaces
terraform workspace list

# Switch to dev
terraform workspace select dev

# Use workspace name in resource naming
```

```hcl
# main.tf — reference workspace in resource names and configs
locals {
  env    = terraform.workspace
  prefix = "${var.project_name}-${local.env}"

  # Per-environment sizing
  instance_configs = {
    dev        = { instance_type = "t3.micro", desired_count = 1 }
    staging    = { instance_type = "t3.small", desired_count = 1 }
    production = { instance_type = "t3.medium", desired_count = 3 }
  }

  current_config = local.instance_configs[local.env]
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.current_config.instance_type

  tags = {
    Name        = "${local.prefix}-app"
    Environment = local.env
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}
```

## Reusable Module Structure

Modules prevent copy-paste infrastructure across environments and projects.

```
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
└── modules/
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── database/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```hcl
# modules/networking/main.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "azs" {
  description = "Availability zones"
  type        = list(string)
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

```hcl
# main.tf — using the module
module "networking" {
  source      = "./modules/networking"
  vpc_cidr    = "10.0.0.0/16"
  environment = terraform.workspace
  azs         = ["us-east-1a", "us-east-1b"]
}

module "compute" {
  source             = "./modules/compute"
  environment        = terraform.workspace
  vpc_id             = module.networking.vpc_id
  private_subnet_ids = module.networking.private_subnet_ids
  instance_type      = local.current_config.instance_type
}
```

## Variables and Secrets

```hcl
# variables.tf
variable "project_name" {
  description = "Project identifier used in resource names"
  type        = string
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true  # redacted from plan output and logs
}
```

```bash
# Environment-specific variable files (committed — no secrets)
# terraform.tfvars is auto-loaded, but per-workspace files need explicit loading

# dev.tfvars
project_name = "myapp"
instance_type = "t3.micro"

# Apply with specific vars file
terraform apply -var-file="dev.tfvars"

# Secrets via environment variables (not committed)
export TF_VAR_db_password="secret-password"
terraform apply -var-file="dev.tfvars"
```

Store actual secret values in AWS SSM Parameter Store or AWS Secrets Manager, and reference them in Terraform:

```hcl
# Read secret from SSM (no plaintext in state)
data "aws_ssm_parameter" "db_password" {
  name = "/myapp/${terraform.workspace}/db_password"
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}
```

## GitHub Actions CI/CD

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'infrastructure/**'
  push:
    branches:
      - main
    paths:
      - 'infrastructure/**'

env:
  TF_VERSION: "1.7.5"

permissions:
  id-token: write   # for OIDC authentication to AWS
  contents: read
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/terraform-github-actions
          aws-region: us-east-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan (staging)
        id: plan
        run: |
          terraform workspace select staging
          terraform plan -var-file="staging.tfvars" -no-color -out=tfplan
        continue-on-error: true

      - name: Post plan to PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const output = `#### Terraform Plan 📖\`${{ steps.plan.outcome }}\`
            <details><summary>Show Plan</summary>
            \`\`\`terraform
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            </details>`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  apply:
    runs-on: ubuntu-latest
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production  # requires manual approval in GitHub
    defaults:
      run:
        working-directory: infrastructure

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/terraform-github-actions
          aws-region: us-east-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Apply (production)
        run: |
          terraform workspace select production
          terraform apply -var-file="production.tfvars" -auto-approve
```

The plan runs on every PR (showing the diff as a comment). The apply runs only on merge to `main`, and the `environment: production` gate requires manual approval from a configured reviewer.

## State Import: Bring Existing Infrastructure Under Control

```bash
# Import existing AWS resources into Terraform state
# First, write the resource config in .tf files, then import

# Import an existing EC2 instance
terraform import aws_instance.app i-1234567890abcdef0

# Import an existing S3 bucket
terraform import aws_s3_bucket.logs my-existing-logs-bucket

# Import an existing RDS instance
terraform import aws_db_instance.main mydb

# After import, run plan — should show no changes if config matches reality
terraform plan -var-file="production.tfvars"
```

## Related Reading

- [AWS Cost Management for Remote Teams](/remote-work-tools/aws-cost-management-remote-teams-guide/)
- [CI/CD Pipeline for Solo Developers: GitHub Actions](/remote-work-tools/ci-cd-pipeline-solo-developer-github-actions/)
- [Home Lab Setup Guide for Remote Developers](/remote-work-tools/home-lab-setup-guide-remote-developers/)

## Related Articles

- [How to Automate DNS Management with Terraform](/remote-work-tools/how-to-automate-dns-management-with-terraform/)
- [Diversity Sourcing Strategy for Remote Teams](/remote-work-tools/remote-team-hiring-diversity-sourcing-strategy-for-distributed-companies/)
- [VS Code Remote Development Setup Guide](/remote-work-tools/vscode-remote-development-setup/)
- [Migrating from AWS CodeCommit to GitHub for Remote Team](/remote-work-tools/migrating-from-aws-codecommit-to-github-for-remote-team-code/)
- [Remote Work Tools: All Guides and Reviews](/remote-work-tools/guides-hub/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Does Teams offer a free tier?**

Most major tools offer some form of free tier or trial period. Check Teams's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.

**How do I get my team to adopt a new tool?**

Start with a small pilot group of willing early adopters. Let them use it for 2-3 weeks, then gather their honest feedback. Address concerns before rolling out to the full team. Forced adoption without buy-in almost always fails.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

{% endraw %}
