# How I Integrated Pass into OpenClaw

A complete guide on setting up secure credential management for AI agents using Unix Pass.

**Original Thread:** https://x.com/GraspOnCrypto/status/2018761493429301352

---

## Overview

This document outlines how I integrated [Pass](https://www.passwordstore.org/) (the Unix Password Store) into my OpenClaw system to securely manage API keys, SSH passphrases, and other secrets. The goal was to ensure credentials are encrypted at rest, never exposed in plaintext, and accessible only when needed.

---

## What is Pass?

Pass is a simple password manager that follows the Unix philosophy:
- Stores passwords in GPG-encrypted files
- Uses standard Unix directory structure
- Works with Git for version control
- Is scriptable and automation-friendly

---

## Prerequisites

Before starting, ensure you have:
- Ubuntu 24 (or similar Linux distribution)
- OpenClaw installed and running
- GPG key pair (preferably without passphrase for automation)
- Git configured

---

## Step-by-Step Integration

### Step 1: Install Pass

```bash
sudo apt update
sudo apt install pass
```

Verify installation:
```bash
pass --version
```

### Step 2: Initialize Pass Store

Initialize with your GPG key:

```bash
# Get your GPG key ID
gpg --list-secret-keys --with-colons | grep fpr | head -1 | cut -d: -f10

# Initialize pass with that key
pass init <YOUR-GPG-KEY-ID>
```

### Step 3: Create Directory Structure

Set up a hierarchical structure for organizing secrets:

```bash
# Create the base directory structure
pass insert openclaw/ai/venice.ai
pass insert openclaw/sshkeys/ed25519
pass insert openclaw/web/moltbook
pass insert openclaw/web/github_pat
```

**Recommended structure:**
```
openclaw/
├── ai/
│   └── venice.ai          # AI service API keys
├── sshkeys/
│   └── ed25519            # SSH key passphrases
└── web/
    ├── moltbook           # Social platform API keys
    └── github_pat         # GitHub Personal Access Token
```

### Step 4: Store Your Secrets

Add your credentials to the pass store:

```bash
# Insert API key (you'll be prompted to paste the secret)
pass insert openclaw/web/moltbook

# Insert SSH passphrase
pass insert openclaw/sshkeys/ed25519

# Insert GitHub PAT
pass insert openclaw/web/github_pat
```

Each command will prompt you to enter the secret value securely.

### Step 5: Create the Pass Skill

Create a custom skill to teach your OpenClaw agent how to use Pass:

1. Create the skill directory:
```bash
mkdir -p ~/.openclaw/skills/pass
```

2. Create `SKILL.md` with:
   - Security policy (TOP SECRET classification)
   - How to retrieve secrets
   - Rules for handling credentials
   - Examples of proper usage

**Key elements of the skill:**
- 🔒 **TOP SECRET PROTOCOL**: All values from Pass are classified as top secret
- **Absolute Rules**: Never mention, never expose, never exfiltrate
- **Permitted Uses**: Direct injection into commands, subprocess env vars
- **Prohibited Uses**: No echo, no logging, no caching

### Step 6: Configure SSH for Non-Interactive Use

To use SSH keys without manual passphrase entry, set up an askpass helper:

```bash
# Create a helper script that retrieves passphrase from Pass
cat > /tmp/ssh_askpass.sh << 'EOF'
#!/bin/bash
pass show openclaw/sshkeys/ed25519
EOF
chmod 700 /tmp/ssh_askpass.sh
```

Add SSH key to agent:
```bash
eval "$(ssh-agent -s)"
DISPLAY="" SSH_ASKPASS="/tmp/ssh_askpass.sh" SSH_ASKPASS_REQUIRE=force ssh-add ~/.ssh/id_ed25519
```

### Step 7: Configure GitHub CLI with Pass

Install and configure `gh` to use your stored PAT:

```bash
# Download and install GitHub CLI
curl -fsSL https://github.com/cli/cli/releases/download/v2.65.0/gh_2.65.0_linux_amd64.tar.gz -o /tmp/gh.tar.gz
tar -xzf /tmp/gh.tar.gz
mkdir -p ~/.local/bin
mv gh_2.65.0_linux_amd64/bin/gh ~/.local/bin/

# Authenticate using Pass
export GH_TOKEN=$(pass show openclaw/web/github_pat)
gh auth status
```

### Step 8: Create Git Repository for Skills

With Pass and GitHub CLI working, create a repo to share your skills:

```bash
# Create local repository
mkdir -p ~/git/skills
cd ~/git/skills
git init
git config user.name "your-username"
git config user.email "your-email@example.com"

# Add remote
git remote add origin git@github.com:your-username/skills.git

# Create initial files (README.md, skill files)
echo "# Skills" > README.md

# Commit and push
 git add -A
git commit -m "Initial commit"
git push -u origin main
```

### Step 9: Maintain Security Discipline

Always follow these rules:

1. **Retrieve only when needed**: `pass show <path>` immediately before use
2. **Use immediately**: Inject directly into commands, don't store in variables
3. **Never expose**: Don't echo, print, or log secret values
4. **Let fall out of scope**: Secrets exist only in memory during execution

**Example - Good:**
```bash
curl -H "Authorization: Bearer $(pass show openclaw/web/moltbook)" https://api.example.com
```

**Example - Bad:**
```bash
API_KEY=$(pass show openclaw/web/moltbook)
echo "Using API key: $API_KEY"  # NEVER DO THIS
```

---

## Verification

Test your setup:

```bash
# 1. Verify Pass works
pass list

# 2. Verify SSH authentication
ssh -T git@github.com

# 3. Verify GitHub CLI
gh auth status

# 4. Verify skill is loaded
ls ~/.openclaw/skills/pass/SKILL.md
```

---

## Benefits of This Setup

- **Encrypted at rest**: All secrets stored with GPG encryption
- **Decrypted only in memory**: Secrets only exist unencrypted during use
- **Reboot-safe**: Works non-interactively after system restart
- **Version controlled**: Pass works with Git for credential history
- **Automation-friendly**: No prompts or manual intervention needed
- **Secure by default**: TOP SECRET policy prevents accidental exposure

---

## Troubleshooting

### SSH Key Not Added Automatically
Ensure `ssh-agent` is running and the askpass helper is set correctly:
```bash
eval "$(ssh-agent -s)"
export SSH_ASKPASS="/tmp/ssh_askpass.sh"
export SSH_ASKPASS_REQUIRE=force
ssh-add ~/.ssh/id_ed25519
```

### Pass Not Found
Make sure `pass` is in your PATH:
```bash
which pass || echo "Pass not installed"
```

### GPG Key Issues
Verify your GPG key is properly set up:
```bash
gpg --list-secret-keys
pass init <key-id>  # Re-initialize if needed
```

---

## Resources

- **Pass Website**: https://www.passwordstore.org/
- **OpenClaw Documentation**: https://docs.openclaw.ai
- **Original Thread**: https://x.com/GraspOnCrypto/status/2018761493429301352
- **Example Skills Repo**: https://github.com/clawdybawty/skills

---

## License

MIT - Share and adapt freely, but keep credentials secure!
