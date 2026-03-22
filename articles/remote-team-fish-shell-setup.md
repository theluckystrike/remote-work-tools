---
layout: default
title: "Remote Team fish Shell Setup Guide"
description: "Share a fish shell config across a distributed team using a dotfiles repo — with shared functions, abbreviations, universal variables, and a one-command bootstrap"
date: 2026-03-22
author: theluckystrike
permalink: /remote-team-fish-shell-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## Remote Team fish Shell Setup Guide

fish (the Friendly Interactive SHell) has autosuggestions, syntax highlighting, and a sane scripting language without the POSIX baggage. For remote teams that work heavily in terminals, sharing a consistent fish config means everyone benefits from the same abbreviations for your internal tools, the same project navigation functions, and the same prompt without anyone spending a day configuring it.

---

## Why fish for Team Config Sharing

- **No `~/.bashrc` sourcing chain**: fish reads `~/.config/fish/` cleanly
- **Universal variables**: set once, persist across all terminals automatically
- **Abbreviations over aliases**: fish abbreviations expand inline so engineers see and learn the full command
- **`fish_plugins` file**: Tide prompt and Fisher plugins can be locked to exact versions

---

## Repo Structure

```
dotfiles/
├── fish/
│   ├── config.fish            -- main config, sourced on startup
│   ├── fish_plugins           -- Fisher plugin lock file (committed)
│   ├── functions/
│   │   ├── mkcd.fish
│   │   ├── proj.fish          -- project switcher
│   │   ├── kdeploy.fish       -- kubectl deploy helper
│   │   └── ghpr.fish          -- open current branch PR
│   ├── conf.d/
│   │   ├── abbreviations.fish -- shared abbreviations
│   │   └── env.fish           -- non-secret env vars
│   └── completions/           -- custom completions for internal tools
└── install.sh
```

---

## Core Config

**`fish/config.fish`**

```fish
# ─── PATH ──────────────────────────────────────────────────────────
fish_add_path /usr/local/bin
fish_add_path $HOME/.local/bin
fish_add_path $HOME/go/bin
fish_add_path $HOME/.cargo/bin

# ─── Options ───────────────────────────────────────────────────────
set -g fish_greeting ""           # no welcome message
set -g EDITOR nvim
set -g VISUAL nvim
set -g PAGER "less -R"

# ─── Colors (Catppuccin Mocha) ─────────────────────────────────────
set -g fish_color_command cba6f7
set -g fish_color_param cdd6f4
set -g fish_color_error f38ba8
set -g fish_color_autosuggestion 585b70
set -g fish_color_comment 6c7086

# ─── History ───────────────────────────────────────────────────────
set -g fish_history_max 10000

# ─── Local config (gitignored) ─────────────────────────────────────
if test -f $HOME/.config/fish/config.local.fish
    source $HOME/.config/fish/config.local.fish
end
```

---

## Shared Abbreviations

**`fish/conf.d/abbreviations.fish`**

```fish
# ─── Git ───────────────────────────────────────────────────────────
abbr -a g    git
abbr -a gs   git status
abbr -a gd   git diff
abbr -a ga   git add
abbr -a gaa  git add -A
abbr -a gc   git commit -m
abbr -a gca  git commit --amend --no-edit
abbr -a gp   git push
abbr -a gpf  git push --force-with-lease
abbr -a gl   git pull --rebase
abbr -a glo  "git log --oneline --graph --decorate -20"
abbr -a gco  git checkout
abbr -a gcb  git checkout -b
abbr -a gst  git stash
abbr -a gsp  git stash pop

# ─── Docker ────────────────────────────────────────────────────────
abbr -a d    docker
abbr -a dc   docker compose
abbr -a dcu  docker compose up -d
abbr -a dcd  docker compose down
abbr -a dcl  docker compose logs -f
abbr -a dps  docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# ─── Kubernetes ────────────────────────────────────────────────────
abbr -a k    kubectl
abbr -a kgp  kubectl get pods
abbr -a kgpa "kubectl get pods -A"
abbr -a kl   kubectl logs -f
abbr -a kex  kubectl exec -it
abbr -a kns  kubectl config set-context --current --namespace
abbr -a kdp  kubectl describe pod

# ─── Misc ──────────────────────────────────────────────────────────
abbr -a ll   "ls -lah"
abbr -a la   "ls -A"
abbr -a ..   "cd .."
abbr -a ...  "cd ../.."
abbr -a h    history
abbr -a v    nvim
abbr -a tf   terraform
abbr -a awsl "aws --profile"
```

---

## Shared Functions

**`fish/functions/proj.fish`** — jump to a project directory:

```fish
function proj --description "Jump to project directory"
    set projects_dir $HOME/projects

    if test (count $argv) -eq 0
        # Interactive picker if fzf is available
        if command -q fzf
            set target (ls $projects_dir | fzf --prompt="project> " --height=40%)
            if test -n "$target"
                cd $projects_dir/$target
                # Auto-activate virtualenv if present
                if test -f .venv/bin/activate.fish
                    source .venv/bin/activate.fish
                end
            end
        else
            ls $projects_dir
        end
    else
        cd $projects_dir/$argv[1]
    end
end
```

**`fish/functions/ghpr.fish`** — open the PR for the current branch:

```fish
function ghpr --description "Open or create PR for current branch"
    set branch (git rev-parse --abbrev-ref HEAD 2>/dev/null)
    if test -z "$branch"
        echo "Not in a git repository"
        return 1
    end

    if test "$branch" = "main" -o "$branch" = "master"
        echo "Already on $branch — check out a feature branch first"
        return 1
    end

    # Use gh CLI to open or create PR
    if command -q gh
        gh pr view --web 2>/dev/null
        or gh pr create --web
    else
        echo "Install gh: brew install gh"
    end
end
```

**`fish/functions/kdeploy.fish`** — deploy to Kubernetes with confirmation:

```fish
function kdeploy --description "Deploy image to Kubernetes deployment"
    # Usage: kdeploy <deployment> <image> [namespace]
    if test (count $argv) -lt 2
        echo "Usage: kdeploy <deployment> <image> [namespace]"
        return 1
    end

    set deployment $argv[1]
    set image $argv[2]
    set ns (if test (count $argv) -ge 3; echo $argv[3]; else; echo "production"; end)

    echo "Deploy $image to $deployment in namespace $ns? [y/N]"
    read confirm
    if test "$confirm" != "y" -a "$confirm" != "Y"
        echo "Aborted"
        return 1
    end

    kubectl set image deployment/$deployment \
        $deployment=$image \
        -n $ns

    kubectl rollout status deployment/$deployment -n $ns
end
```

---

## Fisher Plugin Lock File

**`fish/fish_plugins`** (committed — locks exact plugin versions):

```
jorgebucaran/fisher
PatrickF1/fzf.fish
IlanCosman/tide@v6.1.1
jethrokuan/z
nickeb96/puffer-fish
jorgebucaran/autopair.fish
```

Install plugins from the lock file headlessly:

```bash
fish -c "fisher update"
```

---

## Bootstrap Script

**`install.sh`**

```bash
#!/bin/bash
set -euo pipefail

DOTFILES_DIR="$(cd "$(dirname "$0")" && pwd)"
FISH_CONFIG="${XDG_CONFIG_HOME:-$HOME/.config}/fish"

# Install fish if not present
if ! command -v fish &>/dev/null; then
  if [[ "$(uname)" == "Darwin" ]]; then
    brew install fish
  else
    sudo apt-add-repository ppa:fish-shell/release-3
    sudo apt update && sudo apt install fish
  fi
fi

# Create config dir and symlink
mkdir -p "$FISH_CONFIG/functions" "$FISH_CONFIG/conf.d" "$FISH_CONFIG/completions"

ln -sf "$DOTFILES_DIR/fish/config.fish"            "$FISH_CONFIG/config.fish"
ln -sf "$DOTFILES_DIR/fish/fish_plugins"            "$FISH_CONFIG/fish_plugins"
ln -sf "$DOTFILES_DIR/fish/conf.d/abbreviations.fish" "$FISH_CONFIG/conf.d/abbreviations.fish"
ln -sf "$DOTFILES_DIR/fish/conf.d/env.fish"         "$FISH_CONFIG/conf.d/env.fish"

for fn in "$DOTFILES_DIR/fish/functions/"*.fish; do
  ln -sf "$fn" "$FISH_CONFIG/functions/$(basename "$fn")"
done

# Install Fisher and plugins
fish -c "
  if not functions -q fisher
    curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish | source
    fisher install jorgebucaran/fisher
  end
  fisher update
"

# Set fish as default shell
if ! grep -q "$(which fish)" /etc/shells; then
  which fish | sudo tee -a /etc/shells
fi
chsh -s "$(which fish)"

echo "fish setup complete. Open a new terminal or run: exec fish"
```

---

## Related Reading

- [Remote Team Neovim Setup and Config Sharing](/remote-work-tools/remote-team-neovim-setup-config-sharing/)
- [Remote Team tmux Config Sharing Guide](/remote-work-tools/remote-team-tmux-config-sharing/)
- [How to Set Up Teleport for Secure Access](/remote-work-tools/teleport-secure-access-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
