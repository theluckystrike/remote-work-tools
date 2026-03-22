---
layout: default
title: "How to Automate Remote Server Patching"
description: "Automate OS patching across remote Linux servers with Ansible, unattended-upgrades, and scheduled maintenance windows for distributed teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-remote-server-patching/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Manual patching across dozens of servers is how you miss a critical CVE. Automated patching ensures all servers run current packages, schedules reboots during maintenance windows, and notifies your team of what changed — without manual SSH sessions.

## Key Takeaways

- **Topics covered**: strategy: layers of automation, layer 1: unattended-upgrades (ubuntu), layer 2: ansible full patch playbook
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## Strategy: Layers of Automation

```
Layer 1: unattended-upgrades (security-only, automatic)
Layer 2: Ansible playbook (full patching, scheduled)
Layer 3: Reboot policy (during defined maintenance window)
Layer 4: Notification (Slack alert after patching)
```

## Layer 1: unattended-upgrades (Ubuntu)

Install on every server to handle security patches automatically:

```bash
sudo apt install unattended-upgrades apt-listchanges -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

```ini
# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
  "${distro_id}:${distro_codename}";
  "${distro_id}:${distro_codename}-security";
  "${distro_id}ESMApps:${distro_codename}-apps-security";
  "${distro_id}ESM:${distro_codename}-infra-security";
};

Unattended-Upgrade::Package-Blacklist {
  "docker-ce";
  "docker-ce-cli";
  "postgresql-*";
  "mysql-server";
  // Blacklist packages that need coordinated upgrades
};

Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "false";
Unattended-Upgrade::Automatic-Reboot "false";  // We control reboots
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
Unattended-Upgrade::Mail "ops@example.com";
Unattended-Upgrade::MailReport "on-change";
```

```ini
# /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

```bash
# Test configuration
sudo unattended-upgrade --dry-run --debug

# Run immediately
sudo unattended-upgrade -v
```

## Layer 2: Ansible Full Patch Playbook

```yaml
# playbooks/patch.yml
---
- name: Full system patching
  hosts: "{{ target_hosts | default('all') }}"
  become: true
  serial: "{{ batch_size | default('25%') }}"  # Patch 25% of hosts at a time

  pre_tasks:
    - name: Check if host is in maintenance window
      assert:
        that:
          - ansible_date_time.weekday in ['6', '0']  # Sat or Sun
          - ansible_date_time.hour | int >= 2
          - ansible_date_time.hour | int <= 6
        fail_msg: "Patching only runs during maintenance window (Sat-Sun 02:00-06:00)"
      when: enforce_maintenance_window | default(true) | bool

    - name: Take pre-patch snapshot (if VMware/AWS)
      include_tasks: tasks/snapshot.yml
      when: take_snapshot | default(false) | bool

    - name: Record pre-patch package versions
      shell: dpkg -l | grep -E '^ii' > /tmp/packages-before.txt
      changed_when: false

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 0  # Force fresh

    - name: Get list of upgradeable packages
      command: apt list --upgradeable 2>/dev/null
      register: upgradeable
      changed_when: false

    - name: Show upgradeable packages
      debug:
        msg: "{{ upgradeable.stdout_lines }}"

    - name: Upgrade all packages
      ansible.builtin.apt:
        upgrade: dist
        autoremove: true
        autoclean: true
      register: apt_upgrade

    - name: Check if reboot is required
      stat:
        path: /var/run/reboot-required
      register: reboot_required

  post_tasks:
    - name: Record post-patch package versions
      shell: dpkg -l | grep -E '^ii' > /tmp/packages-after.txt
      changed_when: false

    - name: Calculate changed packages
      shell: diff /tmp/packages-before.txt /tmp/packages-after.txt | grep "^[<>]" | head -50
      register: changed_packages
      changed_when: false

    - name: Reboot if required
      ansible.builtin.reboot:
        reboot_timeout: 300
        msg: "Rebooting after package updates"
      when:
        - reboot_required.stat.exists
        - allow_reboot | default(false) | bool

    - name: Notify Slack
      delegate_to: localhost
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: |
            :white_check_mark: Patching complete on `{{ inventory_hostname }}`
            Changed packages: {{ changed_packages.stdout_lines | length }}
            Reboot required: {{ reboot_required.stat.exists }}
      when: slack_webhook is defined
```

## Running the Patch Playbook

```bash
# Dry run first — see what would change
ansible-playbook playbooks/patch.yml \
  --check --diff \
  -e "target_hosts=webservers" \
  -e "enforce_maintenance_window=false"

# Patch staging (no maintenance window enforcement)
ansible-playbook playbooks/patch.yml \
  -e "target_hosts=staging" \
  -e "enforce_maintenance_window=false" \
  -e "allow_reboot=true" \
  -e "batch_size=50%"

# Patch production during maintenance window
ansible-playbook playbooks/patch.yml \
  -e "target_hosts=production" \
  -e "allow_reboot=true" \
  -e "batch_size=1" \  # One host at a time
  -e "slack_webhook=https://hooks.slack.com/..."

# Patch specific host
ansible-playbook playbooks/patch.yml \
  --limit "web-01.example.com" \
  -e "enforce_maintenance_window=false"
```

## RHEL/CentOS Patching

```yaml
# tasks/patch-rhel.yml
- name: Update all packages (RHEL/CentOS)
  ansible.builtin.dnf:
    name: "*"
    state: latest
    update_cache: true
  register: dnf_update

- name: Install security updates only
  ansible.builtin.dnf:
    name: "*"
    state: latest
    security: true

- name: Check pending kernel updates
  command: needs-restarting -r
  register: needs_restart
  changed_when: false
  failed_when: false  # returns 1 if restart needed
```

## Scheduled Cron Job

```bash
# /etc/cron.d/ansible-patching
# Patch all servers Saturday at 3am UTC
0 3 * * 6 deploy /usr/local/bin/ansible-patching.sh >> /var/log/ansible-patching.log 2>&1
```

```bash
#!/bin/bash
# /usr/local/bin/ansible-patching.sh

set -e

ANSIBLE_DIR="/opt/ansible"
LOG_FILE="/var/log/ansible-patching.log"
SLACK_WEBHOOK="${SLACK_WEBHOOK_PATCHING}"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Starting scheduled patching run"

cd "$ANSIBLE_DIR"

# Update Ansible itself first
pip install -q --upgrade ansible

# Run patching
if ansible-playbook playbooks/patch.yml \
  -e "allow_reboot=true" \
  -e "slack_webhook=${SLACK_WEBHOOK}" \
  >> "$LOG_FILE" 2>&1; then
  log "Patching completed successfully"
else
  log "Patching FAILED"
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-type: application/json" \
    -d '{"text": ":x: Scheduled patching FAILED — check /var/log/ansible-patching.log"}'
  exit 1
fi
```

## Patch Compliance Reporting

```bash
# Generate report: which hosts need patches
cat > playbooks/patch-report.yml << 'EOF'
---
- name: Patch compliance report
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Check upgradeable packages
      command: apt list --upgradeable 2>/dev/null
      register: upgradeable
      changed_when: false
      when: ansible_os_family == "Debian"

    - name: Check security updates available
      command: apt-get -s upgrade 2>&1 | grep "upgraded"
      register: security_check
      changed_when: false
      when: ansible_os_family == "Debian"

    - name: Write host report
      delegate_to: localhost
      lineinfile:
        path: /tmp/patch-report.csv
        line: "{{ inventory_hostname }},{{ ansible_distribution }},{{ ansible_distribution_version }},{{ upgradeable.stdout_lines | length }}"
        create: true
EOF

ansible-playbook playbooks/patch-report.yml
cat /tmp/patch-report.csv | column -t -s,
```

## Related Reading

- [How to Set Up Ansible for Remote Server Management](/remote-work-tools/how-to-set-up-ansible-remote-server-management/)
- [Remote Work Backup Strategy for Developers](/remote-work-tools/remote-work-backup-strategy-for-developers/)
- [Best Practice for Remote Team Escalation Paths](/remote-work-tools/best-practice-for-remote-team-escalation-paths-that-scale-wi/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
