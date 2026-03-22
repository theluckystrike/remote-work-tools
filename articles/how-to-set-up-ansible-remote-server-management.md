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
tags: [remote-work-tools]---

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
# roles/common/tasks/main.yml---
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

# molecule/default/converge.yml
---
- name: Converge
 hosts: all
 become: true
 roles:
 - role: nginx
```

## Idempotency Checks

Always verify idempotency before team rollout:

```bash
# Run twice and confirm no changes on second pass
ansible-playbook playbooks/site.yml | grep -E "changed|failed"
ansible-playbook playbooks/site.yml | grep -E "changed|failed"
# Second run should show: changed=0 failed=0
```

## Related Reading

- [Terraform Remote Team Infrastructure Guide](/remote-work-tools/terraform-remote-team-infrastructure-guide/)
- [Best Secrets Management Tool for Remote Dev Teams](/remote-work-tools/best-secrets-management-tool-for-remote-development-teams-us/)
- [How to Automate Remote Server Patching](/remote-work-tools/how-to-automate-remote-server-patching/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

