---

layout: default
title: "How to Manage Multiple GitHub Accounts for Remote Work"
description: "A practical guide to managing multiple GitHub accounts on a single machine. Learn SSH keys, git config overrides, and workflow patterns for developers working with personal and professional repos."
date: 2026-03-15
author: theluckystrike
permalink: /how-to-manage-multiple-github-accounts-remote-work/
---

{% raw %}
Managing multiple GitHub accounts on a single development machine is a common challenge for developers who work across personal projects, client work, and employment. Whether you are a freelancer handling client repositories or a developer contributing to open source alongside your day job, understanding how to switch between accounts seamlessly will save you time and prevent authentication headaches.

This guide covers practical methods to manage multiple GitHub accounts without constant re-authentication or repository errors.

## Understanding the Core Challenge

When you configure Git with a single username and email, every repository you work with inherits those settings. This creates problems when pushing to repositories under different GitHub accounts. The commit author shows the wrong identity, and you may lack permission to push to repositories owned by your other accounts.

The solution involves SSH keys tied to specific accounts and Git configuration that scopes settings to individual repositories.

## Setting Up SSH Keys for Each Account

SSH keys provide passwordless authentication to GitHub. Each GitHub account needs its own SSH key pair.

Generate a new SSH key for your additional account:

```bash
ssh-keygen -t ed25519 -C "your-personal-email@example.com"
```

When prompted for the file location, use a descriptive name:

```
Enter file in which to save the key (/Users/you/.ssh/id_ed25519): /Users/you/.ssh/id_ed25519_personal
```

Repeat this process for each account, using distinct file names:
- `id_ed25519_work` for your work account
- `id_ed25519_personal` for personal projects
- `id_ed25519_client` for client work

Add each private key to your SSH agent:

```bash
ssh-add ~/.ssh/id_ed25519_work
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_client
```

## Configuring SSH for Multiple Accounts

Edit your SSH config file to map keys to hosts:

```bash
# ~/.ssh/config

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes

Host github-client
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_client
    IdentitiesOnly yes
```

Add the public keys to their corresponding GitHub accounts through Settings → SSH and GPG keys → New SSH key.

## Cloning Repositories with the Correct Identity

When cloning a repository, use the custom host alias instead of the default GitHub URL:

```bash
# Instead of:
git clone git@github.com:organization/repo.git

# Use:
git clone git@github-work:organization/repo.git
```

This tells SSH to use the work key for authentication. For personal repositories:

```bash
git clone git@github-personal:username/my-project.git
```

## Configuring Git Per Repository

For each cloned repository, set the appropriate user identity:

```bash
cd path/to/work-project
git config user.name "Your Name"
git config user.email "work@company.com"

cd path/to/personal-project
git config user.name "Your Name"
git config user.email "personal@gmail.com"
```

These settings stored in `.git/config` override global settings for that specific repository only.

## Using IncludeIf for Directory-Based Configuration

If you organize your projects in separate directories, you can set automatic configuration based on folder location. Add to your global Git config:

```bash
git config --global --includeIf.gitdir:i/path/to/work-projects/.gitconfig path /Users/you/.gitconfig-work
```

Create `/Users/you/.gitconfig-work`:

```ini
[user]
    name = Your Name
    email = work@company.com
```

Create similar files for personal and client directories. Git automatically applies the correct identity when you are inside any directory under those paths.

## A Practical Workflow Example

Consider this directory structure:

```
~/projects/
  ├── work/
  │   ├── client-alpha/
  │   └── internal-tool/
  └── personal/
      ├── open-source-project/
      └── startup-idea/
```

Configure Git globally once, then set directory-based overrides. When you work on anything in `~/projects/work/`, Git uses your work identity automatically. The same applies to personal projects.

To clone a new work repository into the correct structure:

```bash
cd ~/projects/work
git clone git@github-work:company/repo.git
cd repo
# Git already knows this is work-related from the directory
```

## Switching Accounts During Development

Sometimes you need to quickly switch contexts. Create shell aliases for common operations:

```bash
# Add to your .zshrc or .bashrc

alias git-work='git config user.name "Your Name" && git config user.email "work@company.com"'
alias git-personal='git config user.name "Your Name" && git config user.email "personal@gmail.com"'
alias git-show-identity='git config user.name && git config user.email'
```

Run `git-show-identity` anytime to verify your current configuration.

## Troubleshooting Common Issues

If pushes fail or authentication prompts appear, verify your setup:

Check which SSH key Git is attempting to use:

```bash
ssh -vT git@github-work
```

This shows detailed connection information and confirms which key GitHub recognizes.

Ensure your SSH agent has the correct keys loaded:

```bash
ssh-add -l
```

If keys are missing, add them again with `ssh-add`.

## Key Takeaways

Managing multiple GitHub accounts requires setting up separate SSH keys for each account, configuring SSH to use the correct key for each repository, and establishing Git configuration that applies the right identity per project. Directory-based configuration with `includeIf` reduces manual setup when you organize projects by account.

The initial configuration takes about fifteen minutes but pays dividends in saved time and eliminated friction throughout your remote work workflow.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
