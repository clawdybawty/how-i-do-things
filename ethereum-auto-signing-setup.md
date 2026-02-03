# Ethereum Auto-Signing Setup with OpenClaw

A secure walkthrough for setting up an automated Ethereum transaction signing system using OpenClaw, an isolated wallet user, and Foundry Cast.

---

## Overview

This setup creates a secure, isolated environment where:
- **OpenClaw** prepares and submits transactions to an RPC endpoint
- **autoeth** user signs transactions using GPG-protected keys
- **Shared directories** facilitate secure handoff between users
- **Safety mechanisms** (rate limiting, poison pill) prevent unauthorized operations
- **Audit logging** tracks all signing activity

---

## Prerequisites

- Ubuntu 24 (or similar Linux distribution)
- Root or sudo access
- OpenClaw installed and running
- GPG configured

---

## Step-by-Step Setup

### Step 1: Install Required Packages

Install tools for file watching (inotify), JSON processing (jq), and access control (acl):

```bash
sudo apt update
sudo apt install -y inotify-tools jq acl
```

**Package purposes:**
- `inotify-tools`: Watch directories for new transaction files
- `jq`: Parse and manipulate JSON transaction data
- `acl`: Fine-grained access control for shared directories

---

### Step 2: Create and Isolate the Wallet User

Create a dedicated user for transaction signing. This user will:
- Hold GPG-encrypted private keys
- Sign transactions prepared by OpenClaw
- Return signed transactions for submission

```bash
# Create the autoeth user
sudo useradd -m -s /bin/bash autoeth

# Set a strong passphrase
sudo passwd autoeth
```

**Security note:** Use a strong, unique passphrase for this user.

---

### Step 3: Create Shared Directory Structure

Create directories for transaction workflow:

```bash
sudo mkdir -p /var/eth_signing/{pending,signed,completed,rejected,config,logs,control}
```

**Directory purposes:**
- `pending/`: OpenClaw places unsigned transactions here
- `signed/`: autoeth places signed transactions here
- `completed/`: Successfully submitted transactions moved here
- `rejected/`: Invalid or failed transactions moved here
- `config/`: Shared configuration files
- `logs/`: Audit logs and activity records
- `control/`: Runtime control files (rate limits, pause flags)

---

### Step 4: Create Group and Assign Users

Create a shared group and add both users:

```bash
# Create the eth_signers group
sudo groupadd eth_signers

# Add OpenClaw user to the group
sudo usermod -aG eth_signers openclaw

# Add autoeth user to the group
sudo usermod -aG eth_signers autoeth

# Assign ownership of shared directory to root:eth_signers
sudo chown -R root:eth_signers /var/eth_signing
```

---

### Step 5: Set Directory Permissions

Configure permissions for secure shared access:

```bash
# Set SGID bit (2770) - new files inherit group ownership
sudo chmod -R 2770 /var/eth_signing

# Make config, logs, and control world-readable (755)
sudo chmod 755 /var/eth_signing/config
sudo chmod 755 /var/eth_signing/logs
sudo chmod 755 /var/eth_signing/control
```

**Permission breakdown:**
- `2770`: SGID + rwx for owner and group, no access for others
- `755`: rwx for owner, rx for group and others

---

### Step 6: Create Safety and Audit Files

Create control files for rate limiting, poison pill, and audit logging:

```bash
# Create control files
sudo touch /var/eth_signing/control/.rate_limit
sudo touch /var/eth_signing/control/.pause
sudo touch /var/eth_signing/logs/audit.log
```

**File purposes:**
- `.rate_limit`: Tracks signing rate (read-only for signers)
- `.pause`: Poison pill - when non-empty, signing is disabled
- `audit.log`: Immutable append-only log of all signing activity

---

### Step 7: Set File Permissions

Configure specific permissions for control files:

```bash
# Set ownership for control files
sudo chown root:eth_signers /var/eth_signing/control/.rate_limit
sudo chown root:eth_signers /var/eth_signing/control/.pause
sudo chown root:eth_signers /var/eth_signing/logs/audit.log

# Set permissions (read-only for signers)
sudo chmod 644 /var/eth_signing/control/.rate_limit
sudo chmod 644 /var/eth_signing/control/.pause
sudo chmod 644 /var/eth_signing/logs/audit.log

# Make audit.log append-only (immutable)
sudo chattr +a /var/eth_signing/logs/audit.log
```

**Security features:**
- `644`: Root can read/write, others can only read
- `chattr +a`: Audit log can only be appended to, never deleted or overwritten

#### Poison Pill Usage

Pause all signing by writing to the `.pause` file:

```bash
# Pause signing (root only)
echo "paused at $(date)" | sudo tee /var/eth_signing/control/.pause

# Resume signing (root only)
sudo truncate -s 0 /var/eth_signing/control/.pause
```

**Note:** Only root can delete or null the `.pause` file. This is a kill switch - when non-empty, all signing scripts will honor the pause and reject transactions.

---

### Step 8: Install Foundry Cast

Install Cast from Paradigm's Foundry toolkit for Ethereum transaction handling:

```bash
# Install Foundry
curl -L https://foundry.paradigm.xyz | bash

# Reload PATH or open new terminal
source ~/.bashrc

# Install Cast (and other Foundry tools)
foundryup

# Verify installation
cast --version
```

**Cast capabilities:**
- Sign transactions offline
- Parse and format transaction data
- Interact with RPC endpoints
- Encode/decode ABI data

---

## Next Steps

1. **Configure GPG for autoeth**: Set up GPG keypair for the autoeth user
2. **Create signing scripts**: Build scripts that watch `pending/` and output to `signed/`
3. **Implement address blacklist**: Use MyEtherWallet darklist for recipient validation
4. **Set up RPC configuration**: Configure RPC endpoint in `/var/eth_signing/config/`
5. **Create monitoring**: Set up alerts for failed transactions or audit anomalies

---

## Security Considerations

- **Least privilege**: autoeth user can only sign, not submit
- **Audit trail**: All activity logged to append-only file
- **Kill switch**: Root can instantly halt all operations
- **Rate limiting**: Prevents transaction flooding
- **Address blacklist**: Validates recipients against known malicious addresses (https://github.com/MyEtherWallet/ethereum-lists/blob/master/src/addresses/addresses-darklist.json)

---

## Resources

- **Foundry/Cast Documentation**: https://book.getfoundry.sh/cast/
- **MyEtherWallet Darklist**: https://github.com/MyEtherWallet/ethereum-lists/blob/master/src/addresses/addresses-darklist.json
- **inotify-tools**: https://github.com/inotify-tools/inotify-tools
- **Linux ACLs**: https://debian-handbook.info/browse/stable/sect.filesystem-permissions.html

---

## License

MIT - Use at your own risk with cryptocurrency operations.
