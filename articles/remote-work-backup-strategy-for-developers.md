---
layout: default
title: "Remote Work Backup Strategy for Developers"
description: "A practical 3-2-1 backup strategy for remote developers covering dotfiles, code, databases, and cloud sync with automated scripts and verification"
date: 2026-03-22
author: theluckystrike
permalink: /remote-work-backup-strategy-for-developers/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

A developer's backup strategy needs to cover more than documents. Code history lives in git but local work-in-progress, environment configs, credentials managers, and databases need separate protection. This guide builds a 3-2-1 strategy: 3 copies, 2 different media, 1 offsite.

## Table of Contents

- [What Needs Backing Up](#what-needs-backing-up)
- [Dotfiles: Git as Backup](#dotfiles-git-as-backup)
- [macOS: Time Machine + rsync Offsite](#macos-time-machine-rsync-offsite)
- [Linux: Restic to S3/B2](#linux-restic-to-s3b2)
- [Local Database Backups](#local-database-backups)
- [Backup Verification (Critical)](#backup-verification-critical)
- [SSH Keys: Special Handling](#ssh-keys-special-handling)
- [Cloud Sync Is Not a Backup](#cloud-sync-is-not-a-backup)
- [Secrets and Environment Files](#secrets-and-environment-files)
- [Windows and WSL2 Considerations](#windows-and-wsl2-considerations)
- [Testing Your Full Recovery Scenario](#testing-your-full-recovery-scenario)
- [Related Reading](#related-reading)

## What Needs Backing Up

Map your risk before picking tools:

```
Priority 1 — Irreplaceable
  ~/.ssh/                    SSH keys (2FA recovery codes too)
  ~/.gnupg/                  GPG keys
  Password manager vault     (Bitwarden/1Password export)
  ~/Documents/               Non-synced docs
  Work-in-progress code      Uncommitted local branches

Priority 2 — Recoverable but slow
  ~/.config/                 App configs
  ~/.dotfiles/               Shell configs, editor settings
  ~/projects/                Committed code (in git, but local clones)
  Local databases            Dev DBs, SQLite files

Priority 3 — Re-creatable
  node_modules/, .venv/      Dependencies (exclude from backup)
  build/, dist/, .cache/     Build artifacts (exclude)
  ~/Downloads/               Temporary files
```

## Dotfiles: Git as Backup

```bash
# Initialize dotfiles as a bare git repo
git init --bare $HOME/.dotfiles

# Alias for managing dotfiles
echo "alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'" >> ~/.zshrc
source ~/.zshrc

# Hide untracked files (don't show everything in $HOME)
dotfiles config --local status.showUntrackedFiles no

# Add files
dotfiles add ~/.zshrc ~/.gitconfig ~/.vimrc ~/.tmux.conf
dotfiles add ~/.config/nvim/init.lua
dotfiles add ~/.config/alacritty/alacritty.toml

dotfiles commit -m "Add dotfiles"
dotfiles remote add origin git@github.com:yourname/dotfiles.git
dotfiles push -u origin main
```

```bash
# Restore on new machine
git clone --bare git@github.com:yourname/dotfiles.git $HOME/.dotfiles
alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
dotfiles checkout
dotfiles config --local status.showUntrackedFiles no
```

## macOS: Time Machine + rsync Offsite

```bash
# Time Machine: local backup to external drive
# System Settings > Time Machine > Add Backup Disk

# Verify Time Machine is running
tmutil status
# BackupPhase = Copying
# DateOfLatestBackup = 2026-03-22-030000

# Exclude large folders from Time Machine
tmutil addexclusion ~/projects/node_modules
tmutil addexclusion ~/.npm
tmutil addexclusion ~/.cache
tmutil addexclusion ~/Library/Caches

# List exclusions
tmutil isexcluded ~/projects/node_modules
```

```bash
#!/bin/bash
# scripts/offsite-backup.sh
# Runs daily via launchd, syncs to Hetzner storage box

REMOTE_HOST="your-storagebox.your-storagebox.de"
REMOTE_USER="your-username"
REMOTE_PATH="/backup/$(hostname)"
LOG="/var/log/offsite-backup.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG"; }

log "Starting offsite backup"

rsync -avz \
  --delete \
  --exclude='.DS_Store' \
  --exclude='node_modules/' \
  --exclude='.venv/' \
  --exclude='__pycache__/' \
  --exclude='.cache/' \
  --exclude='*.pyc' \
  --backup \
  --backup-dir="$REMOTE_PATH/deleted/$(date +%Y%m%d)" \
  -e "ssh -i ~/.ssh/backup_key -p 23" \
  ~/Documents/ ~/projects/ ~/.ssh/ ~/.config/ \
  "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/files/" \
  >> "$LOG" 2>&1

if [ $? -eq 0 ]; then
  log "Offsite backup succeeded"
else
  log "Offsite backup FAILED"
fi
```

```xml
<!-- ~/Library/LaunchAgents/com.yourname.backup.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.yourname.backup</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/yourname/scripts/offsite-backup.sh</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>2</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>
  <key>RunAtLoad</key>
  <false/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.yourname.backup.plist
```

## Linux: Restic to S3/B2

Restic is the best open-source backup tool — deduplication, encryption, versioning.

```bash
# Install restic
brew install restic          # macOS
sudo apt install restic      # Ubuntu

# Initialize repository (Backblaze B2)
export B2_ACCOUNT_ID="your-account-id"
export B2_ACCOUNT_KEY="your-app-key"
export RESTIC_PASSWORD="your-strong-encryption-password"

restic -r b2:your-bucket-name:backups init

# Or use a local repo for testing
restic -r /Volumes/ExternalDrive/restic-backups init
```

```bash
#!/bin/bash
# scripts/restic-backup.sh

export B2_ACCOUNT_ID="${B2_ACCOUNT_ID}"
export B2_ACCOUNT_KEY="${B2_ACCOUNT_KEY}"
export RESTIC_PASSWORD="${RESTIC_PASSWORD}"
REPO="b2:your-bucket-name:backups"

# Backup
restic -r "$REPO" backup \
  ~/Documents \
  ~/projects \
  ~/.ssh \
  ~/.config \
  --exclude-file ~/.restic-excludes \
  --tag "daily" \
  --verbose

# Forget old snapshots (keep 7 daily, 4 weekly, 12 monthly)
restic -r "$REPO" forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune

# Verify integrity
restic -r "$REPO" check
```

```
# ~/.restic-excludes
node_modules/
.venv/
__pycache__/
*.pyc
.cache/
.npm/
.gradle/
target/
build/
dist/
*.log
.DS_Store
```

## Local Database Backups

```bash
#!/bin/bash
# scripts/backup-local-dbs.sh
# Back up all local development databases

BACKUP_DIR="$HOME/.db-backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p "$BACKUP_DIR"

# PostgreSQL
if command -v pg_dumpall &>/dev/null; then
  pg_dumpall -U postgres | gzip > "$BACKUP_DIR/postgres_all_${DATE}.sql.gz"
  echo "PostgreSQL backed up"
fi

# MySQL
if command -v mysqldump &>/dev/null; then
  mysqldump --all-databases | gzip > "$BACKUP_DIR/mysql_all_${DATE}.sql.gz"
  echo "MySQL backed up"
fi

# SQLite files in projects
find ~/projects -name "*.sqlite" -o -name "*.db" 2>/dev/null | while read f; do
  rel="${f#$HOME/}"
  dest="$BACKUP_DIR/sqlite/${DATE}/${rel}"
  mkdir -p "$(dirname "$dest")"
  cp "$f" "$dest"
done

# Remove backups older than 7 days
find "$BACKUP_DIR" -name "*.gz" -mtime +7 -delete
find "$BACKUP_DIR/sqlite" -mtime +7 -type f -delete

echo "DB backups complete in $BACKUP_DIR"
```

## Backup Verification (Critical)

Backups you haven't tested are worthless. Run monthly:

```bash
#!/bin/bash
# scripts/verify-backups.sh

echo "=== Backup Verification $(date) ==="

# Test restic restore
restic -r "b2:your-bucket-name:backups" \
  restore latest \
  --target /tmp/restore-test \
  --include ~/.zshrc

if [ -f /tmp/restore-test/.zshrc ]; then
  echo "PASS: Restic restore works"
  rm -rf /tmp/restore-test
else
  echo "FAIL: Restic restore failed"
fi

# Verify latest snapshot exists and is recent
LATEST=$(restic -r "b2:your-bucket-name:backups" snapshots --last --json | jq -r '.[0].time')
DAYS_OLD=$(( ($(date +%s) - $(date -d "$LATEST" +%s)) / 86400 ))

if [ "$DAYS_OLD" -lt 2 ]; then
  echo "PASS: Latest backup is ${DAYS_OLD} days old"
else
  echo "FAIL: Latest backup is ${DAYS_OLD} days old — too old!"
fi
```

## SSH Keys: Special Handling

```bash
# Never store raw private keys in cloud sync
# Instead: export encrypted with GPG

gpg --symmetric --cipher-algo AES256 ~/.ssh/id_ed25519
# Creates: ~/.ssh/id_ed25519.gpg

# Store the encrypted version in your git dotfiles or cloud
# Restore:
gpg --decrypt ~/.ssh/id_ed25519.gpg > ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519
```

## Cloud Sync Is Not a Backup

iCloud, Dropbox, and Google Drive sync deletions instantly. If you accidentally `rm -rf ~/projects/critical-work`, the deletion propagates to every device within seconds. These services are useful for active-file access across devices, but they are not backups.

The key distinction: **sync replicates your current state**; **backup preserves historical states**. Use both, and make sure they are independent.

For Dropbox and Google Drive, enable extended version history (Dropbox Plus gives 180 days; Google Drive keeps 30 days of versions). This helps with accidental overwrites but does not protect against ransomware or account compromise.

## Secrets and Environment Files

`.env` files and credential JSON files are the most dangerous things to lose — and the most dangerous to back up carelessly. A structured approach:

```bash
# Audit what secrets you have locally
find ~ -name ".env" -o -name "*.env" -o -name "credentials.json" \
  -o -name "service-account.json" 2>/dev/null | grep -v node_modules | grep -v .git

# For each critical secrets file, store an encrypted copy
# Use age (modern, simpler than GPG) for file encryption

brew install age

# Generate a key pair (store the private key in your password manager)
age-keygen -o ~/.age/key.txt
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aq

# Encrypt a secrets file
age -r age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aq \
  -o ~/.secrets-backup/myproject.env.age \
  ~/projects/myproject/.env

# Decrypt on restore
age -d -i ~/.age/key.txt \
  ~/.secrets-backup/myproject.env.age > ~/projects/myproject/.env
```

Add the `.secrets-backup/` directory to your restic or rsync offsite backup. Keep the age private key in your password manager (1Password, Bitwarden) with a backup export.

## Windows and WSL2 Considerations

Remote developers on Windows using WSL2 have a split filesystem. Back up both sides:

```bash
# WSL2 home directory (inside the Linux layer)
# Access from Windows PowerShell:
# \\wsl$\Ubuntu\home\yourname

# Back up WSL2 with export (creates a tarball)
wsl --export Ubuntu C:\Backups\ubuntu-wsl-$(Get-Date -Format "yyyyMMdd").tar

# Or use restic from inside WSL2 (same script as Linux)
# Install restic in WSL2:
sudo apt install restic

# Windows Documents folder is mounted at /mnt/c/Users/yourname/Documents
# Include it in your restic backup:
restic -r "$REPO" backup \
  ~/Documents \
  /mnt/c/Users/yourname/Documents \
  ~/.ssh \
  ~/.config
```

For Windows-side tooling, WinSCP supports rsync-like sync to remote SSH targets. Robocopy handles local redundancy well:

```powershell
# Mirror Documents to external drive (Windows)
robocopy C:\Users\yourname\Documents E:\Backup\Documents /MIR /R:3 /W:5 /LOG:C:\Logs\backup.log
```

## Testing Your Full Recovery Scenario

Running the verification script monthly is good. Running a full recovery drill quarterly is better. Document the scenario:

```
Recovery Drill Checklist (run on a fresh machine or VM):
[ ] Restore dotfiles from git bare repo
[ ] Decrypt and restore SSH keys from backup
[ ] Restore .env files from encrypted backup
[ ] Clone critical repos (verify SSH keys work)
[ ] Restore dev database from latest backup
[ ] Verify app starts and connects to local DB
[ ] Confirm shell aliases, editor config, git config all present
[ ] Total time to full working environment: ______ minutes
```

The goal is to know, not guess, how long recovery takes. Teams that have done this drill typically discover that their database restore script has a bug, or that a critical .env file was never added to the backup scope. Find these gaps during drills, not during an actual incident.

## Related Reading

- [Best Backup Solutions for Remote Developer Machines](/remote-work-tools/best-backup-solutions-for-remote-developer-machines/)
- [How to Set Up MinIO for Team Object Storage](/remote-work-tools/how-to-set-up-minio-team-object-storage/)
- [Best Dotfiles Manager for Remote Developer Setup](/remote-work-tools/best-dotfiles-manager-for-remote-developer-setup/)

---

## Related Articles

- [Best Backup Solutions for Remote Developer Machines](/remote-work-tools/best-backup-solutions-for-remote-developer-machines/)
- [Manage Dotfiles Across Remote Machines](/remote-work-tools/manage-dotfiles-across-remote-machines/)
- [Backblaze vs CrashPlan for Remote Work Backup](/remote-work-tools/backblaze-vs-crashplan-for-remote-work-backup/)
- [Best Dotfiles Manager for Remote Developer Setup](/remote-work-tools/best-dotfiles-manager-for-remote-developer-setup/)
- [Git Branching Strategy for Remote Teams](/remote-work-tools/git-branching-strategy-remote-teams/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
