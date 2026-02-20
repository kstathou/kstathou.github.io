+++
authors = ["Kostas Stathoulopoulos"]
title = "Managing Multiple GitHub Accounts on One Machine"
date = "2026-02-20"
description = "A complete guide to configuring SSH and Git for seamless multi-account GitHub usage on macOS"
tags = ["github", "ssh", "git", "devtools"]
categories = []
series = []
+++

If you want to write code both for work and personal projects, you've likely faced this challenge: how do you use both your organizational and personal GitHub accounts from the same laptop without constantly switching credentials or accidentally pushing code under the wrong identity?

This guide walks through the complete setup process, and highlights some issues I encountered along the way.

> **Note:** This guide is written for macOS. Linux users will need to omit `UseKeychain yes` from SSH config and use `ssh-add` without the `--apple-use-keychain` flag. The `pbcopy` command can be replaced with `xclip -selection clipboard`. Windows users, you're on your own.

## Understanding the Problem

GitHub requires that each SSH key be associated with exactly one account. This makes sense from a security perspective; your key is your identity. But it creates a practical problem when you have multiple identities, for example, a GitHub account for work and another one for personal projects.

When you run `git push`, two things need to happen correctly. First, SSH needs to authenticate you to GitHub using the right key for that account. Second, Git needs to record the correct author name and email in the commit metadata. These are two completely separate systems, and configuring only one of them leads to confusing behaviour where authentication works but commits appear under the wrong account.

Let's cover below how you can fix this.

## TL;DR — Quick Setup

```bash
# 1. Generate personal key
ssh-keygen -t ed25519 -C "personal@gmail.com" -f ~/.ssh/id_ed25519_personal

# 2. Add keys to agent
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_personal

# 3. Configure SSH (~/.ssh/config)
cat >> ~/.ssh/config << 'EOF'
Host github-personal
    HostName github.com
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes
EOF

# 4. Add public key to GitHub
pbcopy < ~/.ssh/id_ed25519_personal.pub
# → GitHub → Settings → SSH keys → New

# 5. Test
ssh -T git@github-personal

# 6. Clone personal repos with alias
git clone git@github-personal:username/repo.git ~/path/to/personal/repo

# 7. Set up Git identity (see Part 2 for directory-based automation)
```

## Step 1: Setting Up SSH for Multiple Accounts

### Creating Separate Keys

You need distinct SSH key for each GitHub account. If you already have a key for your organization account (check with `ls ~/.ssh/`), you only need to generate one for your personal account:

```bash
ssh-keygen -t ed25519 -C "your-personal-email@gmail.com" -f ~/.ssh/id_ed25519_personal
```

I use Ed25519 rather than RSA because it's the modern standard; smaller keys, better security, and faster operations. The `-f` flag specifies a custom filename so we don't overwrite the existing key.

### Adding Keys to the SSH Agent

The SSH agent manages your keys in memory so you don't have to enter passphrases repeatedly. Add both keys to the agent:

```bash
# Start the agent if not running
eval "$(ssh-agent -s)"

# Add keys (macOS stores passphrase in Keychain)
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_personal
```

Verify the keys are loaded:

```bash
ssh-add -l
```

You should see both keys listed with their fingerprints.

### The SSH Config File

The SSH config file (`~/.ssh/config`) lets you define "host aliases" that map to different keys. When you connect to `github.com`, SSH uses your org key. When you connect to a custom alias like `github-personal`, SSH uses your personal key—even though both ultimately connect to GitHub's servers.

```ssh-config
# Organization GitHub account
Host github.com
    HostName github.com
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519

# Personal GitHub account
Host github-personal
    HostName github.com
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes
```

The `HostName` directive is what makes this work. Even though you're connecting to `github-personal` in your Git commands, SSH knows to actually connect to `github.com`. The alias is just a routing mechanism.

**Why `IdentitiesOnly yes` on the personal config?** This is critical, and Claude Code helped me out here. The SSH agent acts like a keyring that holds all your loaded keys, and by default, SSH tries every key in the agent until one works. If your org key is loaded first and is valid for GitHub, it authenticates before SSH even tries your personal key—meaning both aliases would authenticate as your org account.

`IdentitiesOnly yes` tells SSH to ignore the agent and use only the key specified in `IdentityFile`. This behaviour tripped me up initially: both `ssh -T git@github.com` and `ssh -T git@github-personal` were greeting my org username until I added this.

### Adding Keys to GitHub

Add each public key to its corresponding GitHub account:

1. Copy the key: `pbcopy < ~/.ssh/id_ed25519_personal.pub`
2. Go to GitHub → Settings → SSH and GPG keys → New SSH key
3. Paste and save

Repeat for both accounts (org key to org account, personal key to personal account).

### Testing the Setup

Verify both connections work:

```bash
ssh -T git@github.com          # Should greet your org username
ssh -T git@github-personal     # Should greet your personal username
```

If both show the same username, double-check that `IdentitiesOnly yes` is set for the personal host.

## Step 2: Configuring Git Identity

With SSH working correctly, you might think you're done. But if you make a commit and push it, you'll notice the author email might still be wrong. SSH handles authentication, while Git config controls the metadata embedded in commits.

I started with the manual approach, but I quickly switched to the second one because it is less error-prone.

### The Manual Approach

The simplest solution is to set a global default for your primary account and override it in specific repositories:

```bash
# Global default (org)
git config --global user.name "Your Name"
git config --global user.email "work@company.com"

# Override in personal repos
cd ~/personal-project
git config user.name "Your Name"
git config user.email "personal@gmail.com"
```

This works, but it's error-prone. Every time you clone a personal repo, you need to remember to set the local config. Forget once, and you've got commits with your work email in your personal project's history.

### The Automated Approach: Directory-Based Configuration

Git has a feature called conditional includes that automatically applies different configurations based on where a repository lives. Keep all your work repos in one directory (say, `~/Desktop/work/`) and all personal repos in another (`~/Desktop/personal/`), then tell Git to use different identities for each location.

Edit `~/.gitconfig`:

```ini
[user]
    name = Your Name
    email = work@company.com

[includeIf "gitdir:~/Desktop/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/Desktop/personal/"]
    path = ~/.gitconfig-personal
```

Create `~/.gitconfig-work`:

```ini
[user]
    name = Your Name
    email = work@company.com
```

Create `~/.gitconfig-personal`:

```ini
[user]
    name = Your Name
    email = personal@gmail.com
```

Now Git automatically detects where each repo lives and applies the appropriate identity.

You might wonder why I create a work config file when the global default already has my work email. The answer is future-proofing and clarity. The setup is now symmetric and self-documenting. If I ever change my global default, work repos remain unaffected.

### What About Repos Elsewhere?

What happens if you clone a repo somewhere outside these directories? Git uses the global default. In my setup, that means any repo not in `~/Desktop/personal/` uses my work identity.

If you want to be stricter, remove the global `[user]` section entirely. Git will then refuse to commit in any repo that isn't under a configured directory, forcing you to either move the repo or set local config explicitly. This prevents accidental commits with the wrong identity at the cost of slightly more friction.

## Working on a new repo, for work or a personal project

For organization work:

```bash
git clone git@github.com:company/project.git ~/Desktop/work/project
cd ~/Desktop/work/project
# SSH uses org key, commits use org email
```

For personal projects:

```bash
git clone git@github-personal:username/project.git ~/Desktop/personal/project
cd ~/Desktop/personal/project
# SSH uses personal key, commits use personal email
```

### Converting Existing Repos

If you have a personal repo that was cloned before this setup (using `git@github.com:...`), update the remote URL to use your personal alias:

```bash
cd ~/Desktop/personal/existing-repo
git remote set-url origin git@github-personal:username/existing-repo.git
```

Verify the change:

```bash
git remote -v
```

The key insight from this whole exercise is that SSH and Git are independent systems that both need configuration. SSH handles the "can I connect?" question, while Git handles the "who made this commit?" question.

## Sanity checks

Use these commands to verify your setup is working correctly:

```bash
# Check which SSH keys are loaded in the agent
ssh-add -l

# Test SSH authentication for each account
ssh -T git@github.com
ssh -T git@github-personal

# Debug SSH connection (verbose output shows which key is tried)
ssh -vT git@github-personal

# Check Git identity in current repo
git config user.name
git config user.email

# Check global Git identity
git config --global user.email
```

## Troubleshooting

I stumbled on a few steps along the way, and Claude Code was helpful enough to summarise those for me:

**SSH authenticates as the wrong account**  
Check that `IdentitiesOnly yes` is set for the non-default host. Run `ssh -vT git@github-personal` to see verbose output showing which key SSH attempts to use.

**Commits show the wrong email**  
Verify your directory structure and `includeIf` paths match. Run `git config user.email` inside a repo to see which email Git will use. Remember that `includeIf` paths must end with a trailing slash.

**Keys don't persist after reboot (macOS)**  
Ensure your SSH config includes `AddKeysToAgent yes` and `UseKeychain yes`, and that you added keys with the `--apple-use-keychain` flag.

**"Permission denied (publickey)" error**  
This usually means: (1) the public key isn't added to GitHub, (2) you're using the wrong host alias in the git remote URL, or (3) the SSH agent isn't running. Check with `ssh-add -l` to see loaded keys.
