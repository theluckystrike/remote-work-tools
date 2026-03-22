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

### Step 1: Directory Structure

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

### Step 2: ansible.cfg

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

### Step 3: Inventory File

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

### Step 4: Group Variables

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

### Step 5: Common Role

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

### Step 6: Ansible Vault for Secrets

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

### Step 7: Main Playbook

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

### Step 8: Run Playbooks

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

### Step 9: Ad-Hoc Commands for Teams

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

### Step 10: CI Integration (GitHub Actions)

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

### Step 11: Test Roles with Molecule

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

### Step 12: Idempotency Checks

Always verify idempotency before team rollout:

```bash
# Run twice and confirm no changes on second pass
ansible-playbook playbooks/site.yml | grep -E "changed|failed"
ansible-playbook playbooks/site.yml | grep -E "changed|failed"
# Second run should show: changed=0 failed=0
```

### Step 13: Dynamic Inventory for Cloud Environments

Static inventory files work for fixed infrastructure, but cloud environments with auto-scaling groups require dynamic inventory. Ansible ships plugins for AWS, GCP, and Azure:

```bash
# Install AWS collection
ansible-galaxy collection install amazon.aws

# aws_ec2 inventory plugin configuration
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
    prefix: role
  - key: placement.region
    prefix: region
hostnames:
  - private-ip-address
compose:
  ansible_host: private_ip_address
```

```bash
# Test dynamic inventory
ansible-inventory -i inventory/production/aws_ec2.yml --list

# Use dynamic inventory in playbooks
ansible-playbook -i inventory/production/aws_ec2.yml playbooks/site.yml
```

This approach means newly launched instances automatically appear in the correct host groups based on their tags, without any manual inventory updates.

### Step 14: Ansible Callback Plugins for Better Logging

Remote teams need visibility into what Ansible does across multiple playbook runs. The `profile_tasks` and `log_plays` callback plugins help:

```ini
# ansible.cfg
[defaults]
callback_whitelist = profile_tasks, log_plays, yaml

[callback_log_plays]
log_folder = /var/log/ansible/plays/
```

For central logging, pipe Ansible output to a shared location or use the `ara` (Ansible Run Analysis) tool which provides a web UI showing every task, its result, and the diff:

```bash
pip install ara[server]

# Configure Ansible to use ara callback
export ANSIBLE_CALLBACK_PLUGINS="$(python3 -m ara.setup.callback_plugins)"
export ANSIBLE_ACTION_PLUGINS="$(python3 -m ara.setup.action_plugins)"
export ANSIBLE_LOOKUP_PLUGINS="$(python3 -m ara.setup.lookup_plugins)"

# Start the ara web UI
ara-manage runserver 0.0.0.0:8000
```

With ara, every team member can browse the history of Ansible runs in a browser, inspect task outputs, and compare diffs between runs — no SSH access to the control node required.

### Step 15: Handling Drift in Long-Running Infrastructure

Servers that have been running for months accumulate manual changes that drift from what Ansible expects. The `--check` flag combined with `--diff` produces a drift report:

```bash
# Generate drift report for all servers
ansible-playbook playbooks/site.yml --check --diff 2>&1 | tee drift-report-$(date +%Y%m%d).txt

# Count changed tasks per host
grep "^TASK\|changed:" drift-report-*.txt | grep changed | sort | uniq -c | sort -rn
```

Running this weekly as a scheduled CI job creates an audit trail of infrastructure drift. When the count climbs, it signals that manual changes are accumulating and need to be folded back into roles.

### Step 16: Use Tags for Selective Playbook Runs

Tags let teams run subsets of a playbook without applying the full configuration. This is especially useful in CI pipelines where you want to deploy only the application layer without re-running base OS hardening:

```yaml
# playbooks/site.yml with tags
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - role: common
      tags: [common, base]
    - role: security-hardening
      tags: [security, base]

- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - role: nginx
      tags: [nginx, web]
    - role: app
      tags: [app, deploy]
```

```bash
# Deploy only the app layer
ansible-playbook playbooks/site.yml --tags deploy

# Run everything except security hardening (useful for rapid iteration)
ansible-playbook playbooks/site.yml --skip-tags security

# List all available tags
ansible-playbook playbooks/site.yml --list-tags
```

Tags also help new team members understand which parts of the playbook affect which systems, making the codebase more approachable for engineers who aren't Ansible experts.

### Step 17: Rolling Updates for Zero-Downtime Deployments

Deploying to a fleet of web servers without downtime requires updating servers in batches and checking health before proceeding:

```yaml
# playbooks/deploy.yml
---
- name: Rolling application deploy
  hosts: webservers
  serial: "25%"          # Update 25% of hosts at a time
  max_fail_percentage: 0 # Abort if any host fails
  become: true

  pre_tasks:
    - name: Remove host from load balancer
      community.general.haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        socket: /var/run/haproxy/admin.sock
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"

    - name: Wait for connections to drain
      ansible.builtin.wait_for:
        timeout: 30

  roles:
    - app

  post_tasks:
    - name: Verify app is healthy
      ansible.builtin.uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      retries: 5
      delay: 5

    - name: Re-enable host in load balancer
      community.general.haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        socket: /var/run/haproxy/admin.sock
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"
```

The `serial` parameter controls the batch size — 25% means on an 8-server fleet, Ansible updates 2 servers at a time. If the health check fails on any server in the batch, `max_fail_percentage: 0` halts the entire playbook before the bad deploy reaches the remaining hosts.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [Terraform Remote Team Infrastructure Guide](/remote-work-tools/terraform-remote-team-infrastructure-guide/)
- [Best Secrets Management Tool for Remote Dev Teams](/remote-work-tools/best-secrets-management-tool-for-remote-development-teams-us/)
- [How to Automate Remote Server Patching](/remote-work-tools/how-to-automate-remote-server-patching/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
