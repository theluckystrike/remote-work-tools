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

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Strategy: Layers of Automation

```
Layer 1: unattended-upgrades (security-only, automatic)
Layer 2: Ansible playbook (full patching, scheduled)
Layer 3: Reboot policy (during defined maintenance window)
Layer 4: Notification (Slack alert after patching)
```

### Step 2: Layer 1: unattended-upgrades (Ubuntu)

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

### Step 3: Layer 2: Ansible Full Patch Playbook

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

### Step 4: Run the Patch Playbook

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

### Step 5: RHEL/CentOS Patching

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

### Step 6: Scheduled Cron Job

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

### Step 7: Handling Reboot Coordination Across Distributed Teams

Rebooting production servers across multiple time zones without notice is how outages happen at 4am for someone. Build a reboot coordination workflow:

```yaml
# playbooks/reboot-notify.yml
---
- name: Coordinate reboot with team
  hosts: localhost
  tasks:
    - name: Post reboot notice to Slack
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: |
            :warning: *Scheduled reboot in 30 minutes*
            Hosts: {{ groups[target_group] | join(', ') }}
            Window: {{ ansible_date_time.date }} {{ ansible_date_time.hour }}:{{ ansible_date_time.minute }} UTC
            Reason: Post-patch kernel update
            Owner: {{ lookup('env', 'USER') }}
            React with :white_check_mark: to acknowledge or :x: to delay.
      when: slack_webhook is defined

    - name: Wait for acknowledgement window
      pause:
        seconds: 1800  # 30 minutes
        prompt: "Press Enter to proceed with reboots, Ctrl+C to abort"
```

For fully automated overnight patching, skip the pause and rely on the maintenance window enforcement in the playbook to prevent accidental daytime reboots.

### Step 8: Inventory Management for Heterogeneous Fleets

Real fleets mix Ubuntu, RHEL, Debian, and Amazon Linux. Structure your inventory to handle this cleanly:

```ini
# inventory/production.ini
[webservers]
web-01.example.com  ansible_python_interpreter=/usr/bin/python3
web-02.example.com  ansible_python_interpreter=/usr/bin/python3

[dbservers]
db-01.example.com   patch_priority=critical
db-02.example.com   patch_priority=critical

[monitoring]
grafana-01.example.com  reboot_ok=false  # Never auto-reboot monitoring

[ubuntu:children]
webservers

[rhel:children]
dbservers

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key
enforce_maintenance_window=true
allow_reboot=false
```

```yaml
# group_vars/ubuntu.yml
patch_manager: apt
kernel_update_pkg: linux-image-generic

# group_vars/rhel.yml
patch_manager: dnf
kernel_update_pkg: kernel
```

This structure lets you run the same playbook across mixed OS environments without conditionals scattered throughout the tasks.

### Step 9: Kernel Live Patching for Zero-Downtime Security Fixes

For servers that cannot tolerate any reboot, kernel live patching applies security fixes to the running kernel without a restart. On Ubuntu:

```bash
# Enable Canonical Livepatch
sudo snap install canonical-livepatch
sudo canonical-livepatch enable <your-token>

# Check live patch status
sudo canonical-livepatch status --verbose
```

On RHEL/CentOS with kpatch:

```bash
# Install kpatch
sudo dnf install kpatch

# List available patches
sudo kpatch list

# Load a patch (no reboot required)
sudo kpatch load /usr/lib/kpatch/$(uname -r)/kpatch-*.ko

# Make persistent across reboots
sudo kpatch install /usr/lib/kpatch/$(uname -r)/kpatch-*.ko
```

Live patching does not replace traditional patching — it handles critical CVEs between maintenance windows, not a permanent substitute. Schedule full reboots quarterly even for live-patched servers to apply accumulated package updates.

### Step 10: Integrate Patch Status with Your Monitoring Stack

Patching without observability means you do not know when it breaks something. Push patch results to your monitoring:

```bash
# Push patch metrics to Prometheus pushgateway
push_metric() {
  local HOST=$1
  local PACKAGES_UPDATED=$2
  local REBOOT_REQUIRED=$3

  cat <<EOF | curl -s --data-binary @- \
    "http://pushgateway.example.com:9091/metrics/job/ansible_patching/instance/${HOST}"
ansible_last_patch_timestamp $(date +%s)
ansible_packages_updated_total ${PACKAGES_UPDATED}
ansible_reboot_required ${REBOOT_REQUIRED}
EOF
}

# Call from your patching script after completion
push_metric "web-01" "23" "0"
```

In Grafana, build a "Patch Compliance" dashboard with:
- Hosts patched in the last 7 days (green)
- Hosts awaiting reboot (yellow)
- Hosts not patched in 30+ days (red alert)

Set an alert on the red panel that fires to `#ops` if any production host exceeds 30 days without a patch run. This gives your security team a live compliance view without manual spreadsheet updates.

### Step 11: Test Patches in a Staging Pipeline

Never patch production without a staging run. Add a sequential pipeline:

```bash
#!/bin/bash
# patch-pipeline.sh — run staging first, then prod after validation

set -e

echo "=== Patching staging ==="
ansible-playbook playbooks/patch.yml \
  -e "target_hosts=staging" \
  -e "enforce_maintenance_window=false" \
  -e "allow_reboot=true"

echo "=== Running smoke tests against staging ==="
./scripts/smoke-test.sh staging.example.com
SMOKE_RESULT=$?

if [ $SMOKE_RESULT -ne 0 ]; then
  echo "Staging smoke tests FAILED — aborting production patching"
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-type: application/json" \
    -d '{"text":":x: Patch pipeline aborted — staging smoke tests failed. Production patching skipped."}'
  exit 1
fi

echo "=== Staging clean — patching production ==="
ansible-playbook playbooks/patch.yml \
  -e "target_hosts=production" \
  -e "allow_reboot=true" \
  -e "batch_size=1"
```

This pattern catches kernel incompatibilities, application crashes after library upgrades, and config file changes introduced by package updates before they hit your production servers.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [How to Set Up Ansible for Remote Server Management](/remote-work-tools/how-to-set-up-ansible-remote-server-management/)
- [Remote Work Backup Strategy for Developers](/remote-work-tools/remote-work-backup-strategy-for-developers/)
- [Best Practice for Remote Team Escalation Paths](/remote-work-tools/best-practice-for-remote-team-escalation-paths-that-scale-wi/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
