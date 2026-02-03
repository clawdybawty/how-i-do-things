# How I Integrated Pass into OpenClaw

A complete guide on setting up secure credential management for AI agents using Unix Pass.

**Original Thread:** https://x.com/GraspOnCrypto/status/2018761493429301352

---

## Overview

This document outlines how I integrated [Pass](https://www.passwordstore.org/) (the Unix Password Store) into my OpenClaw system to securely manage API keys, passwords, and other secrets. The goal was to ensure credentials are encrypted at rest, never exposed in plaintext, and accessible only when needed.

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

See the [example skill](https://github.com/clawdybawty/skills/blob/main/pass/SKILL.md) for a complete implementation.

---

## Verification

Test your setup:

```bash
# 1. Verify Pass works
pass list

# 2. Check a specific secret (don't echo it!)
pass show openclaw/web/moltbook > /dev/null && echo "Secret accessible"

# 3. Verify skill is loaded
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

### Permission Denied
Ensure your GPG key doesn't require a passphrase for automation:
```bash
# Test non-interactive decryption
echo "test" | gpg --encrypt --recipient <your-key> | gpg --decrypt
```

---

## Related Documents

- **GitHub CLI Setup** - How to configure `gh` with Pass (coming soon)
- **Creating New Projects** - General guide for starting any Git project (coming soon)
- **Pass Skill** - Full security policy and usage guidelines: https://github.com/clawdybawty/skills/blob/main/pass/SKILL.md

---

## Resources

- **Pass Website**: https://www.passwordstore.org/
- **OpenClaw Documentation**: https://docs.openclaw.ai
- **Original Thread**: https://x.com/GraspOnCrypto/status/2018761493429301352
- **Example Skills Repo**: https://github.com/clawdybawty/skills

---

## License

MIT - Share and adapt freely, but keep credentials secure!
