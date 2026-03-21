---
layout: default
title: "Manage Dotfiles Across Remote Machines"
description: "Set up dotfile management with GNU Stow or Chezmoi to sync shell configs, editor settings, and aliases across remote servers, laptops, and dev containers."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /manage-dotfiles-across-remote-machines/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Dotfiles are the configuration files in your home directory that control your terminal, editor, aliases, and tools. When you work across multiple remote servers, a home machine, and a laptop, keeping these in sync manually is tedious and error-prone. A dotfile manager turns your config into a Git repository you can clone anywhere.

This guide covers two approaches: GNU Stow (simple, no dependencies beyond git) and Chezmoi (more powerful, handles secrets and machine-specific config).

## The Bare Git Repo Approach (No Extra Tools)

Before reaching for a tool, consider the bare git repo approach. It uses only git — no additional software needed:

```bash
# Initialize a bare git repo in ~/.dotfiles
git init --bare $HOME/.dotfiles

# Create an alias to run git against this bare repo
# (using 'dotfiles' instead of 'git')
alias dotfiles='git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'

# Don't show untracked files (your home dir has thousands)
dotfiles config --local status.showUntrackedFiles no

# Add the alias to your .bashrc/.zshrc
echo "alias dotfiles='git --git-dir=\$HOME/.dotfiles/ --work-tree=\$HOME'" >> ~/.bashrc

# Add remote and push
dotfiles remote add origin git@github.com:yourname/dotfiles.git
dotfiles push -u origin main
```

**Track and commit files:**

```bash
# Track a config file
dotfiles add ~/.bashrc
dotfiles add ~/.vimrc
dotfiles add ~/.config/nvim/init.lua
dotfiles add ~/.tmux.conf
dotfiles add ~/.gitconfig
dotfiles add ~/.config/alacritty/alacritty.toml

# Commit and push
dotfiles commit -m "dotfiles: initial commit"
dotfiles push

# Check status
dotfiles status

# View diff
dotfiles diff ~/.bashrc
```

**Bootstrap on a new machine:**

```bash
# On a fresh machine
git clone --bare git@github.com:yourname/dotfiles.git $HOME/.dotfiles

alias dotfiles='git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'

dotfiles checkout
# If there are conflicts (files already exist), back them up:
# mkdir -p ~/.dotfiles-backup && dotfiles checkout 2>&1 | grep -E "\s+\." | awk {'print $1'} | xargs -I{} mv {} ~/.dotfiles-backup/{}
# Then run dotfiles checkout again

dotfiles config --local status.showUntrackedFiles no
```

## GNU Stow

GNU Stow manages dotfiles by creating symlinks from your home directory to files in a structured `~/dotfiles` folder. Each program gets its own subdirectory.

```bash
# Install
sudo apt-get install stow    # Debian/Ubuntu
brew install stow             # macOS

# Create dotfiles repo structure
mkdir -p ~/dotfiles/{bash,zsh,vim,nvim,tmux,git,alacritty}

# Move your config files into the repo
mv ~/.bashrc ~/dotfiles/bash/.bashrc
mv ~/.bash_profile ~/dotfiles/bash/.bash_profile
mv ~/.vimrc ~/dotfiles/vim/.vimrc
mv ~/.tmux.conf ~/dotfiles/tmux/.tmux.conf
mv ~/.gitconfig ~/dotfiles/git/.gitconfig

# For files in subdirectories, mirror the structure
mkdir -p ~/dotfiles/nvim/.config/nvim
mv ~/.config/nvim/init.lua ~/dotfiles/nvim/.config/nvim/init.lua

mkdir -p ~/dotfiles/alacritty/.config/alacritty
mv ~/.config/alacritty/alacritty.toml ~/dotfiles/alacritty/.config/alacritty/

# Stow creates symlinks from ~ back to ~/dotfiles/
cd ~/dotfiles
stow bash     # creates ~/.bashrc → ~/dotfiles/bash/.bashrc
stow vim
stow tmux
stow git
stow nvim
stow alacritty

# Verify symlinks
ls -la ~/.bashrc
# ~/.bashrc -> /home/user/dotfiles/bash/.bashrc
```

**Bootstrap script:**

```bash
#!/bin/bash
# bootstrap.sh — run this on a new machine
set -e

DOTFILES_REPO="git@github.com:yourname/dotfiles.git"
DOTFILES_DIR="$HOME/dotfiles"

# Install stow if missing
if ! command -v stow &> /dev/null; then
  sudo apt-get update && sudo apt-get install -y stow
fi

# Clone if not already present
if [ ! -d "$DOTFILES_DIR" ]; then
  git clone "$DOTFILES_REPO" "$DOTFILES_DIR"
fi

cd "$DOTFILES_DIR"

# List of packages to stow (directories in dotfiles repo)
PACKAGES=(bash zsh vim nvim tmux git alacritty)

for pkg in "${PACKAGES[@]}"; do
  if [ -d "$pkg" ]; then
    echo "Stowing $pkg..."
    stow --adopt "$pkg" 2>/dev/null || stow "$pkg"
  fi
done

echo "Dotfiles installed."
```

## Chezmoi

Chezmoi is a full-featured dotfile manager. It handles secrets (via 1Password, Bitwarden, or age encryption), machine-specific configuration (different settings on Linux vs macOS), and templates.

```bash
# Install chezmoi
sh -c "$(curl -fsLS get.chezmoi.io)"

# Or
brew install chezmoi

# Initialize with your repo
chezmoi init git@github.com:yourname/dotfiles.git

# Add files to chezmoi management
chezmoi add ~/.bashrc
chezmoi add ~/.tmux.conf
chezmoi add ~/.gitconfig
chezmoi add ~/.config/nvim/init.lua

# View what chezmoi would change
chezmoi diff

# Apply changes
chezmoi apply

# Edit a managed file (opens in $EDITOR, then applies)
chezmoi edit ~/.bashrc
```

**Machine-specific config with templates:**

```bash
# ~/.local/share/chezmoi/dot_gitconfig.tmpl
# chezmoi templates use Go template syntax

[user]
  name = {{ .name }}
  email = {{ .email }}

[core]
  editor = {{ if eq .chezmoi.os "darwin" }}code --wait{{ else }}nvim{{ end }}

[credential]
  helper = {{ if eq .chezmoi.os "darwin" }}osxkeychain{{ else }}store{{ end }}
```

Configure the template variables in `~/.config/chezmoi/chezmoi.toml`:

```toml
[data]
  name = "Your Name"
  email = "you@example.com"
```

**Secrets with 1Password:**

```bash
# In a chezmoi template file (.tmpl extension)
# Reference a 1Password secret

[api_credentials]
  token = {{ onepasswordRead "op://Personal/GitHub PAT/token" }}
  key = {{ onepasswordRead "op://Work/AWS/secret_access_key" }}
```

**Bootstrap on a new machine:**

```bash
# One-liner: install chezmoi + apply dotfiles
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply git@github.com:yourname/dotfiles.git
```

## Handling Machine-Specific Differences

Common differences between machines: work vs personal email in gitconfig, Linux vs macOS paths, work VPN config that shouldn't be in a public repo.

```bash
# Stow: use separate packages per machine type
~/dotfiles/
├── bash/           — shared
├── vim/            — shared
├── git-personal/   — stow on personal machine
├── git-work/       — stow on work machine
└── macos/          — macOS only

# Bootstrap script selects based on hostname or env var
if [[ "$HOSTNAME" == *"work"* ]]; then
  stow git-work
else
  stow git-personal
fi

# Chezmoi: use .chezmoi.hostname or custom data
# ~/.local/share/chezmoi/dot_gitconfig.tmpl
[user]
{{ if .work_machine }}
  email = you@company.com
{{ else }}
  email = you@personal.com
{{ end }}
```

## Sync Workflow

```bash
# After changing a config file, push the update
# With bare git:
dotfiles add ~/.tmux.conf
dotfiles commit -m "tmux: add popup window keybind"
dotfiles push

# With stow: edit the file in ~/dotfiles/ directly (symlink means changes are immediate)
vim ~/dotfiles/tmux/.tmux.conf
cd ~/dotfiles && git add -A && git commit -m "tmux: add popup keybind" && git push

# With chezmoi:
chezmoi edit ~/.tmux.conf  # opens, applies on save
cd $(chezmoi source-path) && git add -A && git commit -m "tmux: add popup keybind" && git push

# On other machines: pull and apply
# bare git: dotfiles pull
# stow: git pull (symlinks already in place)
# chezmoi: chezmoi update
```


## Related Articles

- [permission-matrix.yaml](/remote-work-tools/how-to-manage-client-access-permissions-across-remote-team-t/)
- [How to Manage Remote Journalism Team Across International](/remote-work-tools/how-to-manage-remote-journalism-team-across-international-bu/)
- [How to Manage Remote Team Across More Than 8 Timezones Guide](/remote-work-tools/how-to-manage-remote-team-across-more-than-8-timezones-guide/)
- [How to Manage Remote Team Handoffs Across Time Zones: A](/remote-work-tools/how-to-manage-remote-team-handoffs-across-time-zones/)
- [Best Dotfiles Manager for Remote Developer Setup](/remote-work-tools/best-dotfiles-manager-for-remote-developer-setup/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
