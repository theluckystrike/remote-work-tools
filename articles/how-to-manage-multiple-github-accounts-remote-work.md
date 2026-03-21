---
layout: default
title: "How to Manage Multiple GitHub Accounts for Remote Work"
description: "Learn practical methods to manage multiple GitHub accounts on one machine using SSH keys and Git configuration. Perfect for developers handling"
date: 2026-03-15
author: theluckystrike
permalink: /how-to-manage-multiple-github-accounts-remote-work/
categories: [guides]
tags: [remote-work-tools, github, ssh, git, remote-work, productivity]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
# How to Manage Multiple GitHub Accounts for Remote Work

Managing multiple GitHub accounts on a single machine is a common challenge for developers working on personal projects alongside client work or full-time employment. Whether you maintain a personal repository, contribute to open-source projects, and push code to a corporate organization—all from the same laptop—this guide covers the practical setup you need.

The core solution involves generating separate SSH keys for each account and configuring Git to use the right identity based on the repository you're working with. Here's how to set this up from scratch.

## Generating SSH Keys for Each Account

First, generate a unique SSH key for each GitHub account. Avoid using the default key for everything—separate keys give you granular control over which account accesses which repository.

Generate a key for your personal account:

```bash
ssh-keygen -t ed25519 -C "your-personal-email@example.com"
```

When prompted, save this key with a descriptive name:

```
Enter file in which to save the key (/Users/you/.ssh/id_ed25519): /Users/you/.ssh/github_personal
```

Repeat the process for your work account:

```bash
ssh-keygen -t ed25519 -C "your-work-email@company.com"
```

Save this one as:

```
Enter file in which to save the key (/Users/you/.ssh/id_ed25519): /Users/you/.ssh/github_work
```

This creates four files for each key: the private key (github_personal) and the public key (github_personal.pub), plus the same for your work key.

## Adding Keys to the SSH Agent

Start the SSH agent and add your keys:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/github_personal
ssh-add ~/.ssh/github_work
```

To avoid running these commands every session, add them to your shell profile:

```bash
echo 'eval "$(ssh-agent -s)"' >> ~/.zshrc
echo 'ssh-add ~/.ssh/github_personal' >> ~/.zshrc
echo 'ssh-add ~/.ssh/github_work' >> ~/.zshrc
```

## Configuring SSH for Account Routing

Edit your SSH config file to route connections based on the host:

```bash
vim ~/.ssh/config
```

Add the following configuration:

```
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_personal
    IdentitiesOnly yes

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_work
    IdentitiesOnly yes
```

The `IdentitiesOnly yes` setting ensures SSH uses only the specified key, preventing authentication failures from trying the wrong key.

## Adding Public Keys to GitHub

Copy each public key and add it to the corresponding GitHub account:

```bash
# Personal account key
cat ~/.ssh/github_personal.pub
# Copy output and add to GitHub > Settings > SSH Keys

# Work account key
cat ~/.ssh/github_work.pub
# Copy output and add to work GitHub organization
```

In GitHub, go to Settings → SSH and GPG keys → New SSH key, paste the public key, and save.

## Cloning Repositories with the Right Identity

When cloning repositories, use the custom host alias instead of the default github.com:

```bash
# Clone using your personal identity
git clone git@github-personal:username/repo.git

# Clone using your work identity
git clone git@github-work:organization/repo.git
```

This works because SSH reads your config and selects the correct key based on the host alias.

## Configuring Git Per Repository

For existing repositories, set the remote URL to use the appropriate host alias:

```bash
cd your-project-directory
git remote set-url origin git@github-work:company/project.git
```

Verify the change:

```bash
git remote -v
# Output: origin  git@github-work:company/project.git (fetch)
# Output: origin  git@github-work:company/project.git (push)
```

## Setting Git User Identity Per Repository

Configure Git user details specifically for each repository:

```bash
cd path/to/work/project
git config user.name "Your Name"
git config user.email "your-work-email@company.com"

cd path/to/personal/project
git config user.name "Your Name"
git config user.email "your-personal-email@example.com"
```

This ensures commits show the correct author based on which account you're using. The local config overrides your global settings for that specific repository.

## Using Git Config Includes for Cleaner Setup

For a more organized approach, use Git config includes. Create a separate config file for each account:

```bash
# ~/.ssh/gitconfig-personal
[user]
    name = Your Name
    email = your-personal-email@example.com

# ~/.ssh/gitconfig-work
[user]
    name = Your Name
    email = your-work-email@company.com
```

Then reference these files in your global Git config:

```bash
git config --global includeIf.gitdir:~/personal/.path "~/gitconfig-personal"
git config --global includeIf.gitdir:~/work/.path "~/gitconfig-work"
```

This automatically applies the correct identity based on which directory you're in.

## Verifying Your Setup

Test that SSH connections work for each account:

```bash
ssh -T git@github-personal
# Expected: Hi username! You've successfully authenticated...

ssh -T git@github-work
# Expected: Hi username! You've successfully authenticated...
```

If you see "Permission denied" or authentication failures, double-check that the public key is added to the correct GitHub account and that your SSH config points to the right key file.

## Troubleshooting Common Issues

Wrong account on commits: Run `git log` to check commit authors. If incorrect, amend the last commit with `git commit --amend --author="Name <email>"` or rebase to fix multiple commits.

SSH key not being used: Verify the key is added to the agent with `ssh-add -l`. If empty, add the keys again. Check file permissions—SSH requires private keys to be readable only by you: `chmod 600 ~/.ssh/github_personal`.

Wrong key offered to GitHub: The `IdentitiesOnly yes` setting in your SSH config prevents SSH from offering multiple keys. Without it, SSH tries keys in order until one works, which can cause delays or failures with certain repository permissions.

## When to Use HTTPS Instead

Some environments block SSH port 22. In these cases, configure Git to use HTTPS with a credential helper:

```bash
git config --global url."https://github-personal/".insteadOf "git@github-personal:"
git config --global url."https://github-work/".insteadOf "git@github-work:"
```

This approach uses your GitHub personal access token stored in the credential helper, avoiding SSH entirely.



## Related Articles

- [How to Manage Remote Team When Multiple Parents Have](/remote-work-tools/how-to-manage-remote-team-when-multiple-parents-have-overlap/)
- [How to Manage Multiple Freelance Clients Effectively](/remote-work-tools/how-to-manage-multiple-freelance-clients-effectively/)
- [Generate weekly team activity report from GitHub](/remote-work-tools/how-to-manage-hybrid-team-where-some-members-are-fully-remot/)
- [How to Manage Work-Life Balance as a Remote Developer](/remote-work-tools/how-to-manage-work-life-balance-remote-developer/)
- [Best Business Bank Accounts for Freelancers 2026: A](/remote-work-tools/best-business-bank-accounts-for-freelancers-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
