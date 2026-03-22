---
layout: default
title: "Remote Team tmux Config Sharing Guide"
description: "Share a common tmux config across a distributed team using a dotfiles repo, tpm plugin lock files, and session templates that new engineers can bootstrap in minutes"
date: 2026-03-22
author: theluckystrike
permalink: /remote-team-tmux-config-sharing/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}
## Remote Team tmux Config Sharing Guide

Every engineer on a distributed team SSHing into servers has their own tmux setup, which means half your sessions use `Ctrl-b` and half use `Ctrl-a`, window navigation differs person to person, and pair programming feels awkward until someone copies a config they found in a gist three years ago. A shared tmux config in your dotfiles repo solves this — one command to install, the same layout for everyone, with room for personal overrides.

---

## Repo Structure

```
dotfiles/
├── tmux/
│   ├── tmux.conf            -- shared team config (committed)
│   ├── tmux.conf.local      -- gitignored personal overrides
│   ├── plugins/             -- tpm plugin bundle (if vendored)
│   └── sessions/
│       ├── dev.sh           -- development session template
│       └── infra.sh         -- infra monitoring session template
└── install.sh               -- bootstrap script
```

---

## The Shared Config

**`tmux/tmux.conf`**

```bash
# ─── Prefix key ───────────────────────────────────────
# Use Ctrl-a (like screen). Ctrl-b is too hard to reach on most keyboards.
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# ─── Core options ─────────────────────────────────────
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:Tc"  # true color support
set -g history-limit 50000
set -g mouse on
set -g base-index 1           # windows start at 1
setw -g pane-base-index 1     # panes start at 1
set -g renumber-windows on    # close window 2, window 3 becomes window 2
set -g escape-time 10         # faster ESC key response (important for Neovim)
set -g focus-events on

# ─── Status bar ───────────────────────────────────────
set -g status-position top
set -g status-style "bg=#1e1e2e,fg=#cdd6f4"
set -g status-left " #[bold]#S #[nobold]│ "
set -g status-left-length 30
set -g status-right "#[fg=#a6adc8]%H:%M  #[fg=#cdd6f4]#{=21:pane_title} "
set -g status-right-length 50

set -g window-status-format " #I:#W "
set -g window-status-current-format " #[bold]#I:#W#{?window_zoomed_flag, 🔍,} "
set -g window-status-current-style "fg=#cba6f7"

# ─── Pane splitting ───────────────────────────────────
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}"  # new window inherits cwd

# ─── Pane navigation (vim-like) ───────────────────────
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Also support Alt+arrow without prefix
bind -n M-Left  select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up    select-pane -U
bind -n M-Down  select-pane -D

# ─── Pane resizing ────────────────────────────────────
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# ─── Window navigation ────────────────────────────────
bind -n S-Left  previous-window
bind -n S-Right next-window
bind Tab last-window  # toggle between last two windows

# ─── Copy mode (vi keys) ──────────────────────────────
set -g mode-keys vi
bind Enter copy-mode
bind -T copy-mode-vi v   send-keys -X begin-selection
bind -T copy-mode-vi y   send-keys -X copy-pipe-and-cancel "pbcopy 2>/dev/null || xclip -selection clipboard"
bind -T copy-mode-vi C-v send-keys -X rectangle-toggle

# ─── Session management ───────────────────────────────
bind S choose-session   # pick a session from list
bind N new-session      # new session
bind R command-prompt "rename-session '%%'"

# ─── Reload config ────────────────────────────────────
bind r source-file ~/.config/tmux/tmux.conf \; display "Config reloaded"

# ─── Plugins (managed by tpm) ─────────────────────────
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-resurrect'   # save/restore sessions
set -g @plugin 'tmux-plugins/tmux-continuum'   # auto-save sessions
set -g @plugin 'tmux-plugins/tmux-yank'        # cross-platform copy

set -g @continuum-restore 'on'
set -g @resurrect-capture-pane-contents 'on'

# ─── Local overrides ──────────────────────────────────
if-shell "[ -f ~/.config/tmux/tmux.conf.local ]" \
  "source-file ~/.config/tmux/tmux.conf.local"

# Initialize tpm (keep this last)
run '~/.tmux/plugins/tpm/tpm'
```

---

## Bootstrap Script

**`install.sh`** (idempotent — safe to run multiple times):

```bash
#!/bin/bash
set -euo pipefail

DOTFILES_DIR="${DOTFILES_DIR:-$(dirname "$(realpath "$0")")}"
TMUX_CONFIG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/tmux"

# Create config directory
mkdir -p "$TMUX_CONFIG_DIR"

# Symlink the shared config
ln -sf "$DOTFILES_DIR/tmux/tmux.conf" "$TMUX_CONFIG_DIR/tmux.conf"

# Copy local overrides template (don't overwrite if already exists)
if [[ ! -f "$TMUX_CONFIG_DIR/tmux.conf.local" ]]; then
  cp "$DOTFILES_DIR/tmux/tmux.conf.local.example" \
     "$TMUX_CONFIG_DIR/tmux.conf.local"
  echo "Created $TMUX_CONFIG_DIR/tmux.conf.local — customize as needed"
fi

# Install tpm
if [[ ! -d "$HOME/.tmux/plugins/tpm" ]]; then
  git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
fi

# Install plugins headlessly
tmux new-session -d -s "setup" 2>/dev/null || true
~/.tmux/plugins/tpm/bin/install_plugins
tmux kill-session -t setup 2>/dev/null || true

echo "tmux setup complete. Start a new session with: tmux new -s work"
```

---

## Session Templates

Pre-defined session layouts so the team starts with consistent workspace structure:

**`tmux/sessions/dev.sh`**

```bash
#!/bin/bash
# dev.sh — standard development session
# Usage: bash tmux/sessions/dev.sh [project-path]

SESSION="dev"
PROJECT="${1:-$HOME/projects/myapp}"

# Exit if session exists
tmux has-session -t "$SESSION" 2>/dev/null && tmux attach -t "$SESSION" && exit

tmux new-session -d -s "$SESSION" -c "$PROJECT" -n "editor"

# Window 1: Editor
tmux send-keys -t "${SESSION}:editor" "nvim ." Enter

# Window 2: Dev server split — top runs server, bottom runs terminal
tmux new-window -t "$SESSION" -n "server" -c "$PROJECT"
tmux split-window -t "${SESSION}:server" -v -p 30
tmux send-keys -t "${SESSION}:server.1" "npm run dev" Enter

# Window 3: Git
tmux new-window -t "$SESSION" -n "git" -c "$PROJECT"
tmux send-keys -t "${SESSION}:git" "git status" Enter

# Window 4: Tests (split: watch mode left, test output right)
tmux new-window -t "$SESSION" -n "tests" -c "$PROJECT"
tmux split-window -t "${SESSION}:tests" -h -p 40

# Focus editor
tmux select-window -t "${SESSION}:editor"
tmux attach -t "$SESSION"
```

---

## Personal Overrides Template

**`tmux/tmux.conf.local.example`**

```bash
# tmux.conf.local — personal overrides, NOT committed to the repo
# Copy to ~/.config/tmux/tmux.conf.local and customize

# Personal prefix preference
# set -g prefix C-b
# unbind C-a
# bind C-b send-prefix

# Different status bar color
# set -g status-style "bg=#282828,fg=#ebdbb2"

# Personal aliases / bindings
# bind G new-window -c "~" \; send-keys "lazygit" Enter

# Larger history
# set -g history-limit 100000
```

---

## Shared tmux Over SSH with `tmate`

For async pair debugging, share a read-only session URL:

```bash
# Install tmate
brew install tmate  # macOS
# or: sudo apt install tmate

# Start a shared session
tmate new-session -s shared

# Get the SSH connection string to share
tmate show-messages
# ssh ro-xxxxx@nyc1.tmate.io  (read-only)
# ssh rw-xxxxx@nyc1.tmate.io  (read-write)
```

---

## Keeping Plugins in Sync Across the Team

The biggest drift point in shared tmux configs is plugin versions. Two engineers on different machines have different versions of `tmux-resurrect` and saved sessions stop restoring correctly. Pin versions in the config:

```bash
# tmux.conf — pin plugin commits, not just repos
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible#v2.0.0'
set -g @plugin 'tmux-plugins/tmux-resurrect#v4.0.0'
set -g @plugin 'tmux-plugins/tmux-continuum#v3.0.0'
set -g @plugin 'tmux-plugins/tmux-yank#v5.0.0'
```

Alternatively, vendor the plugins directly in the dotfiles repo:

```bash
# install.sh additions — vendor tpm and plugins
mkdir -p "$DOTFILES_DIR/tmux/plugins"

# Clone/update each plugin at a pinned commit
clone_or_update() {
  local repo="$1" target="$2" commit="$3"
  if [[ -d "$target" ]]; then
    git -C "$target" fetch --quiet
    git -C "$target" checkout "$commit" --quiet
  else
    git clone --quiet "$repo" "$target"
    git -C "$target" checkout "$commit" --quiet
  fi
}

clone_or_update \
  https://github.com/tmux-plugins/tpm \
  "$DOTFILES_DIR/tmux/plugins/tpm" \
  "b840227"  # tag: v3.1.0

# Point tpm at vendored directory
# In tmux.conf:
# run '~/.config/tmux/plugins/tpm/tpm'
```

Then symlink the vendored plugin directory instead of using tpm's git-based installer. New engineers get the exact same versions without a network fetch.

---

## Infra Session Template for On-Call

A dedicated session layout for incident response keeps the monitoring and debugging context separate from normal dev work:

**`tmux/sessions/infra.sh`**

```bash
#!/bin/bash
# infra.sh — on-call / infra session
SESSION="infra"

tmux has-session -t "$SESSION" 2>/dev/null && tmux attach -t "$SESSION" && exit

tmux new-session -d -s "$SESSION" -n "logs"

# Window 1: Log tailing split
# Top: app logs, Bottom: system logs
tmux send-keys -t "${SESSION}:logs" \
  "ssh deploy@app-01.internal 'tail -f /var/log/app/app.log'" Enter
tmux split-window -t "${SESSION}:logs" -v -p 30
tmux send-keys -t "${SESSION}:logs.2" \
  "ssh deploy@app-01.internal 'journalctl -f -u myapp'" Enter

# Window 2: Metrics — Prometheus queries
tmux new-window -t "$SESSION" -n "metrics"
tmux send-keys -t "${SESSION}:metrics" \
  "watch -n5 'curl -s http://prometheus.internal:9090/api/v1/query \
  --data-urlencode \"query=rate(http_requests_total[5m])\" | jq .'" Enter

# Window 3: Kubernetes
tmux new-window -t "$SESSION" -n "k8s"
tmux split-window -t "${SESSION}:k8s" -h
tmux send-keys -t "${SESSION}:k8s.1" "watch kubectl get pods -n production" Enter
tmux send-keys -t "${SESSION}:k8s.2" "kubectl events -n production --watch" Enter

# Window 4: SSH + runbook
tmux new-window -t "$SESSION" -n "shell"

tmux select-window -t "${SESSION}:logs"
tmux attach -t "$SESSION"
```

---

## Related Reading

- [Remote Team Neovim Setup and Config Sharing](/remote-work-tools/remote-team-neovim-setup-config-sharing/)
- [Remote Team fish Shell Setup Guide](/remote-work-tools/remote-team-fish-shell-setup/)
- [How to Set Up Teleport for Secure Access](/remote-work-tools/teleport-secure-access-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
