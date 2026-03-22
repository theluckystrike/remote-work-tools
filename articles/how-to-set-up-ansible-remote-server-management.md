---
layout: default
title: "How to Set Up Ansible for Remote Server Management"
description: "Configure Ansible to manage remote servers securely with inventory files, roles, and vaults for distributed infrastructure teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-ansible-remote-server-management/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Ansible lets remote teams manage hundreds of servers without manual SSH sessions. This guide covers a production-ready setup: inventory structure, roles, vaults for secrets, and CI integration so your distributed team can push config changes safely.

## Prerequisites

- Python 3.8+ on the control node
- SSH access to target servers (key-based)
- Ansible 2.14+

```bash
pip install ansible ansible-lint
ansible --version
# ansible [core 2.14.x]
```

## Directory Structure

A clean layout keeps roles reusable across projects.

```
ansible/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── webservers.yml
│   └── staging/
│       └── hosts.yml
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/
│   ├── nginx/
│   └── postgres/
├── playbooks/
│   ├── site.yml
│   ├── deploy.yml
│   └── patch.yml
└── vault/
    └── secrets.yml
```

## ansible.cfg

```ini
[defaults]
inventory          = inventory/production
remote_user        = deploy
private_key_file   = ~/.ssh/id_ed25519
host_key_checking  = False
retry_files_enabled = False
stdout_callback    = yaml
gathering          = smart
fact_caching       = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400

[ssh_connection]
pipelining         = True
ssh_args           = -o ControlMaster=auto -o ControlPersist=60s
```

## Inventory File

```yaml
# inventory/production/hosts.yml
all:
  children:
    webservers:
      hosts:
        web-01.example.com:
          ansible_host: 10.0.1.10
        web-02.example.com:
          ansible_host: 10.0.1.11
    dbservers:
      hosts:
        db-01.example.com:
          ansible_host: 10.0.2.10
          postgres_version: "15"
    monitoring:
      hosts:
        mon-01.example.com:
          ansible_host: 10.0.3.10
```

## Group Variables

```yaml
# inventory/production/group_vars/all.yml
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org

syslog_server: 10.0.3.10
deploy_user: deploy
ssh_port: 22

# inventory/production/group_vars/webservers.yml
nginx_worker_processes: auto
nginx_worker_connections: 1024
app_port: 8080
```

## Common Role

```yaml
# roles/common/tasks/main.yml
---
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Install common packages
  ansible.builtin.package:
    name:
      - curl
      - git
      - htop
      - unzip
      - fail2ban
      - ufw
    state: present

- name: Set timezone
  community.general.timezone:
    name: UTC

- name: Configure NTP
  ansible.builtin.template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
    owner: root
    group: root
    mode: '0644'
  notify: restart ntp

- name: Create deploy user
  ansible.builtin.user:
    name: "{{ deploy_user }}"
    shell: /bin/bash
    groups: sudo
    append: true
    create_home: true

- name: Add SSH authorized key for deploy user
  ansible.posix.authorized_key:
    user: "{{ deploy_user }}"
    key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    state: present
```

```yaml
# roles/common/handlers/main.yml
---
- name: restart ntp
  ansible.builtin.service:
    name: ntp
    state: restarted

- name: reload ufw
  community.general.ufw:
    state: reloaded
```

## Ansible Vault for Secrets

Never store plaintext credentials in git.

```bash
# Create vault password file (outside repo)
echo "your-strong-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass

# Create encrypted secrets file
ansible-vault create vault/secrets.yml --vault-password-file ~/.vault_pass

# Edit existing vault
ansible-vault edit vault/secrets.yml --vault-password-file ~/.vault_pass
```

```yaml
# vault/secrets.yml (encrypted at rest)
db_password: "s3cur3-db-pass"
api_key: "sk-xxxx-yyyy-zzzz"
smtp_password: "mail-secret"
```

Add to ansible.cfg:

```ini
[defaults]
vault_password_file = ~/.vault_pass
```

Reference vault variables in tasks:

```yaml
- name: Configure database connection
  ansible.builtin.template:
    src: database.conf.j2
    dest: /etc/app/database.conf
    mode: '0600'
  vars:
    password: "{{ db_password }}"
```

## Main Playbook

```yaml
# playbooks/site.yml
---
- name: Apply common configuration to all servers
  hosts: all
  become: true
  vars_files:
    - ../vault/secrets.yml
  roles:
    - common

- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - nginx
    - app

- name: Configure database servers
  hosts: dbservers
  become: true
  roles:
    - postgres
```

## Running Playbooks

```bash
# Check syntax before running
ansible-lint playbooks/site.yml

# Dry run (check mode)
ansible-playbook playbooks/site.yml --check --diff

# Run against staging first
ansible-playbook -i inventory/staging playbooks/site.yml

# Run only specific tags
ansible-playbook playbooks/site.yml --tags "nginx,common"

# Limit to single host
ansible-playbook playbooks/site.yml --limit web-01.example.com

# Run against production with verbose output
ansible-playbook playbooks/site.yml -v
```

## Ad-Hoc Commands for Teams

Quick operations without full playbooks:

```bash
# Check disk usage across all web servers
ansible webservers -m shell -a "df -h /"

# Restart nginx on all web servers
ansible webservers -m service -a "name=nginx state=restarted" --become

# Copy a file to all servers
ansible all -m copy -a "src=/tmp/cert.pem dest=/etc/ssl/cert.pem mode=0644" --become

# Run a shell command and collect output
ansible all -m shell -a "uptime" -o

# Gather facts about a host
ansible web-01.example.com -m setup | grep ansible_distribution
```

## CI Integration (GitHub Actions)

```yaml
# .github/workflows/ansible.yml
name: Ansible Deploy

on:
  push:
    branches: [main]
    paths:
      - 'ansible/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Ansible
        run: pip install ansible ansible-lint

      - name: Write vault password
        run: echo "${{ secrets.VAULT_PASSWORD }}" > ~/.vault_pass && chmod 600 ~/.vault_pass

      - name: Write SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519

      - name: Lint playbooks
        run: ansible-lint ansible/playbooks/site.yml

      - name: Run check mode
        run: ansible-playbook ansible/playbooks/site.yml --check --diff

      - name: Deploy to production
        run: ansible-playbook ansible/playbooks/site.yml
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False"
```

## Testing Roles with Molecule

```bash
pip install molecule molecule-docker

# Initialize molecule in a role
cd roles/nginx
molecule init scenario --driver-name docker

# Run full test cycle
molecule test
```

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  roles:
    - role: nginx
```

Molecule runs your role inside a Docker container and verifies it applies without errors. Add a verify step to assert the intended state:

```yaml
# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Check nginx is running
      ansible.builtin.service_facts:

    - name: Assert nginx is active
      ansible.builtin.assert:
        that:
          - "'nginx' in services"
          - "services['nginx'].state == 'running'"
        fail_msg: "nginx is not running after role apply"
```

This gives remote team members a way to verify infrastructure changes locally in Docker before pushing to staging — no need for a live server.

## Idempotency Checks

Always verify idempotency before team rollout:

```bash
# Run twice and confirm no changes on second pass
ansible-playbook playbooks/site.yml | grep -E "changed|failed"
ansible-playbook playbooks/site.yml | grep -E "changed|failed"
# Second run should show: changed=0 failed=0
```

## Dynamic Inventory for Cloud Infrastructure

Static `hosts.yml` files break when servers are provisioned and destroyed automatically. For AWS environments, use the EC2 dynamic inventory plugin:

```bash
pip install boto3 botocore
ansible-galaxy collection install amazon.aws
```

```yaml
# inventory/production/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - eu-west-1
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: ""
    separator: ""
  - key: placement.region
    prefix: region_
compose:
  ansible_host: private_ip_address
```

Test and use dynamic inventory:

```bash
# List all hosts Ansible would discover
ansible-inventory -i inventory/production/aws_ec2.yml --list

# Run against a dynamically discovered group
ansible tag_Role_webserver -i inventory/production/aws_ec2.yml -m ping

# Run a playbook against auto-discovered webservers
ansible-playbook -i inventory/production/aws_ec2.yml playbooks/site.yml \
  --limit tag_Role_webserver
```

Add a bastion host for servers without public IPs:

```ini
# ansible.cfg ssh_connection section
[ssh_connection]
pipelining    = True
ssh_args      = -o ProxyJump=bastion.example.com -o ControlMaster=auto -o ControlPersist=60s
```

## AWX for Team-Wide Playbook Execution

For larger remote teams, running playbooks from individual laptops causes consistency issues — different Python versions, different vault passwords in different locations. AWX (open-source Ansible Tower) centralizes execution with a web UI and API:

```bash
# Deploy AWX with Docker Compose
git clone https://github.com/ansible/awx.git
cd awx
docker-compose -f tools/docker-compose/_sources/docker-compose.yml up -d
# Access the UI at http://localhost:8013 (admin/password)
```

Key AWX capabilities for distributed teams:

| Feature | Benefit for Remote Teams |
|---------|--------------------------|
| Credentials vault | SSH keys and vault passwords stored centrally, not on laptops |
| Job templates | Locked playbook+inventory combinations prevent drift |
| RBAC | Control who runs which playbooks against which environments |
| Job history | Full audit log: who ran what, when, with what output |
| Webhooks | Trigger playbooks automatically from GitHub merges |
| Notifications | Slack or email alerts on job success or failure |

A typical team workflow: engineer opens a PR with playbook changes, CI runs `ansible-lint` and `--check` mode, team reviews the diff in the PR, and on merge AWX automatically triggers the playbook against production via GitHub webhook.

---

## Tagging Tasks for Selective Runs

As your playbook library grows, running the full `site.yml` on every change becomes slow. Use tags to run only the relevant parts:

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
  tags: [nginx, install]

- name: Configure nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: reload nginx
  tags: [nginx, config]

- name: Enable nginx service
  ansible.builtin.service:
    name: nginx
    enabled: true
    state: started
  tags: [nginx, service]
```

With tags, a config change can be applied in seconds rather than running the full playbook:

```bash
# Apply only nginx config changes — skip install and service tasks
ansible-playbook playbooks/site.yml --tags "nginx,config"

# Patch common packages across all servers without touching app config
ansible-playbook playbooks/site.yml --tags "install" --limit all

# Skip database tasks entirely during a web-only deploy
ansible-playbook playbooks/site.yml --skip-tags "postgres"
```

For remote teams, tagging is especially valuable because it enables teammates in different time zones to apply targeted fixes without needing to understand the full playbook tree.

---

## Related Reading

- [Terraform Remote Team Infrastructure Guide](/remote-work-tools/terraform-remote-team-infrastructure-guide/)
- [Best Secrets Management Tool for Remote Dev Teams](/remote-work-tools/best-secrets-management-tool-for-remote-development-teams-us/)
- [How to Automate Remote Server Patching](/remote-work-tools/how-to-automate-remote-server-patching/)
- [Async Decision-Making Framework for Remote Teams](/remote-work-tools/articles/how-to-set-up-async-decision-making-framework-guide/)

---

## Related Articles

- [Best SSH Key Management Solution for Distributed Remote](/remote-work-tools/best-ssh-key-management-solution-for-distributed-remote-engi/)
- [VS Code Remote Development Setup Guide](/remote-work-tools/vscode-remote-development-setup/)
- [Linux Server Hardening Guide for Remote Developers](/remote-work-tools/linux-server-hardening-remote-developers/)
- [Remote Work Security Hardening Checklist](/remote-work-tools/remote-work-security-hardening-checklist/)
- [Best API Key Management Workflow for Remote Development](/remote-work-tools/best-api-key-management-workflow-for-remote-development-team/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
