# 🔐 Secure SSH Setup & Access Management Guide

This document provides a complete, step-by-step guide for setting up **secure SSH access** to an Ubuntu server (such as AWS EC2), including **CI/CD automation user setup**. It covers generating keys, connecting to the server, creating sudo users, configuring permissions, passwordless sudo for automation, and applying best practices for security.

---

## 🎯 Quick Start: CI/CD Automation User Setup

For immediate CI/CD automation setup, run this script on your server:

```bash
#!/bin/bash
# Complete CI/CD User Setup Script

NEW_USER="emma"  # Change to your preferred username

# Create user with home directory
sudo useradd -m -s /bin/bash $NEW_USER

# Add to sudo group
sudo usermod -aG sudo $NEW_USER

# Set password (optional, skip for key-only access)
# sudo passwd $NEW_USER

# Copy SSH keys from ubuntu user
sudo rsync --archive --chown=${NEW_USER}:${NEW_USER} ~/.ssh /home/${NEW_USER}/

# Fix SSH permissions
sudo chmod 700 /home/${NEW_USER}/.ssh
sudo chmod 600 /home/${NEW_USER}/.ssh/authorized_keys
sudo chown -R ${NEW_USER}:${NEW_USER} /home/${NEW_USER}/.ssh

# Enable passwordless sudo for automation (SAFER method)
echo "${NEW_USER} ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/${NEW_USER}
sudo chmod 440 /etc/sudoers.d/${NEW_USER}

echo "✅ CI/CD user '${NEW_USER}' setup complete!"
echo "🔑 User can login via SSH keys and run sudo commands without password"
```

---

## 🧩 1. Generate a Local SSH Key (if you don't have one)

Run this on your **local machine** (Linux/Mac/WSL):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
or
ssh-keygen 
```

* Press **Enter** to save to the default location: `~/.ssh/id_ed25519`
* Optionally set a **passphrase** for extra protection.

**Files created:**

| File                    | Description                                      |
| ----------------------- | ------------------------------------------------ |
| `~/.ssh/id_ed25519`     | 🔒 Private key (keep secret, never share)        |
| `~/.ssh/id_ed25519.pub` | 🔓 Public key (safe to share or copy to servers) |

---

## 🌍 2. Initial Login to Server (EC2 / Cloud)

If it's your **first time connecting**, use the `.pem` key provided by your cloud host (e.g., AWS):

```bash
ssh -i ~/Downloads/your-aws-key.pem ubuntu@<SERVER_IP>
```

**Notes:**
* Replace `<SERVER_IP>` with your actual instance IP.
* Default user for Ubuntu EC2: **ubuntu**
* Direct root login is disabled for security — use `sudo` for admin commands.

---

## 💻 3. Basic SSH Command Format

Once your SSH key is added, connect simply using:

```bash
ssh username@<SERVER_IP>
```

Or if you want to specify a custom key:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<SERVER_IP>
```

**Example:**

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@13.218.36.173
```

---

## 🗂️ 4. Prepare `.ssh` Folder on the Server

If it doesn't exist, create it (after connecting with your `.pem` key):

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

This ensures the folder exists and has correct permissions.

---

## 🔑 5. Add Your Local Public Key to the Server

You have **two options**:

### ✅ Option 1 — Manual Copy

**On your local machine:**

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the full line.

**On the server:**

```bash
echo "PASTE_YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

✅ This appends your new key without removing the existing `.pem` key.

---

### ⚡ Option 2 — Automatic Copy with `ssh-copy-id`

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=~/Downloads/your-aws-key.pem" ubuntu@<SERVER_IP>
```

This logs in with the `.pem` file and installs your local SSH key automatically.

---

## 🧪 6. Test Passwordless Login

From your local machine:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<SERVER_IP>
```

If successful, you'll log in without using the `.pem` file.

---

## 👤 7. Create a New Sudo User (Recommended)

It's best practice to use a **dedicated user** for daily operations.

### 🔧 Method A: For Interactive Users (with password)

```bash
sudo adduser emma
sudo usermod -aG sudo emma
```

This method prompts for a password and creates home directory automatically.

### 🔧 Method B: For CI/CD Automation (key-based, no password)

```bash
# Create user without password prompt
sudo useradd -m -s /bin/bash emma
sudo usermod -aG sudo emma

# Copy SSH access
sudo rsync --archive --chown=emma:emma ~/.ssh /home/emma/

# Fix permissions for SSH
sudo chmod 700 /home/emma/.ssh
sudo chmod 600 /home/emma/.ssh/authorized_keys
sudo chown -R emma:emma /home/emma/.ssh
```

Now test login as the new user from your local machine:

```bash
ssh emma@<SERVER_IP>
```

✅ You now have your own sudo user with SSH access.

---

## 🤖 8. CI/CD Automation Setup

### 🔑 Passwordless Sudo Configuration

#### **Understanding `sudo visudo` vs `/etc/sudoers.d/emma`**

This is a **common point of confusion**. Let me clarify carefully:

---

### **A. `sudo visudo`**

* Opens the main sudoers file safely (`/etc/sudoers`) in an editor.
* You can add lines like:

```text
emma ALL=(ALL) NOPASSWD:ALL
```

* Pros:
  * Central place to manage sudo rules.
  * Safe: `visudo` **checks syntax** before saving to prevent breaking sudo.
* Cons:
  * All sudo rules in **one file**; can get messy with multiple users or automation scripts.
  * Mistakes here can lock you out of `sudo`.

---

### **B. `/etc/sudoers.d/emma`**

* This is a **separate file just for emma**.
* Using the command:

```bash
echo "emma ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/emma
```

* Pros:
  * Cleaner: each user or service can have its **own sudo file**.
  * Easier to manage, especially for multiple automation users.
  * `sudo` automatically includes all files in `/etc/sudoers.d/`.
  * Safer: mistakes only affect that user, not global sudoers.
* Cons:
  * You need to ensure correct file permissions:

```bash
sudo chmod 440 /etc/sudoers.d/emma
```

> **Important:** `sudoers.d` files must be readable by root only (`440`) and cannot be writable by normal users. Otherwise, `sudo` will refuse to work.

---

### ✅ TL;DR: Which Method to Use?

| Method                     | Where it writes   | Best for                                             |
| -------------------------- | ----------------- | ---------------------------------------------------- |
| `sudo visudo`              | `/etc/sudoers`    | Single admin changes, global rules                   |
| `/etc/sudoers.d/emma` file | `/etc/sudoers.d/` | **Automation users**, separate configs, safer management |

---

💡 **Rule of thumb:**

* For **CI/CD automation users**, always use `/etc/sudoers.d/username` — it's cleaner, easier to manage, and safer.
* Use `visudo` for **global or complex rules** affecting multiple users.

---

### 🔧 Implementation Examples

**Method 1: Using `/etc/sudoers.d/` (Recommended for CI/CD)**

```bash
# Create dedicated sudoers file for emma
echo "emma ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/emma
sudo chmod 440 /etc/sudoers.d/emma

# Verify it works
sudo -U emma -l
```

**Method 2: Using `visudo`**

```bash
# Edit main sudoers file
sudo visudo

# Add this line at the end:
# emma ALL=(ALL) NOPASSWD:ALL
```

**Method 3: Using `visudo` with separate file (BEST PRACTICE)**

```bash
# Safely create/edit emma's sudoers file
sudo visudo -f /etc/sudoers.d/emma

# Add the line and save
```

### 🛡️ Security Considerations for Automation

**For production, restrict sudo privileges instead of using ALL:**

```bash
# Only allow specific commands without password
echo "emma ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart nginx,/usr/bin/apt update,/usr/bin/git" | sudo tee /etc/sudoers.d/emma
sudo chmod 440 /etc/sudoers.d/emma
```

### 🔄 CI/CD Pipeline Integration

**GitHub Actions Example:**
```yaml
- name: Deploy to Server
  uses: appleboy/ssh-action@v0.1.3
  with:
    host: ${{ secrets.SERVER_IP }}
    username: emma
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      cd /var/www/app
      sudo git pull
      sudo systemctl restart nginx
```

**Required Secrets in your CI/CD platform:**
- `SERVER_IP`: Your server's IP address
- `SSH_PRIVATE_KEY`: Emma's private SSH key

---

## ⚙️ 9. Optional: SSH Config for Convenience

Create or edit your local SSH config file:

```bash
nano ~/.ssh/config
```

Add this block:

```
Host myserver
    HostName <SERVER_IP>
    User emma
    IdentityFile ~/.ssh/id_ed25519

Host foudawire
    HostName <SERVER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

Then connect easily using:

```bash
ssh myserver
ssh foudawire
```

No need to type IP or key path anymore.

---

## 🔒 10. Optional: Disable Root & Password Login

After confirming your new user works perfectly, secure SSH further:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Then restart the SSH service:

```bash
sudo systemctl restart ssh
```

**⚠️ Warning:** Only do this after confirming SSH key login works!

---

## 🧪 11. Testing Your CI/CD Setup

**Test SSH key login:**
```bash
ssh emma@<SERVER_IP>
```

**Test passwordless sudo:**
```bash
ssh emma@<SERVER_IP> "sudo whoami"
# Should return "root" without password prompt
```

**Test specific automation commands:**
```bash
ssh emma@<SERVER_IP> "sudo systemctl status nginx"
```

**Verify sudoers configuration:**
```bash
# Check what sudo privileges emma has
sudo -U emma -l
```

---

## 🧠 12. Best Practices

### ✅ General Security
- **Never share** your private key
- Keep your `.pem` file safe and read-only (`chmod 400`)
- Use **one key per user or device**
- Rotate keys regularly (update `~/.ssh/authorized_keys`)
- Use `sudo` for admin tasks — never log in directly as root
- Make backups of your `.ssh` folder

### ✅ Sudo Configuration
- Prefer `/etc/sudoers.d/` over main sudoers file for user-specific rules
- Always use `visudo` for editing to prevent syntax errors
- Set correct permissions (`440`) on sudoers.d files
- Restrict sudo privileges to only necessary commands in production

### ✅ CI/CD Specific
- Store SSH private keys in secure secrets/vaults
- Use dedicated service accounts for different applications
- Regularly audit sudo privileges
- Monitor automation user activity
- Consider using SSH certificates for enhanced security

### ✅ Server Hardening
- Enable fail2ban for SSH protection
- Use non-standard SSH ports (optional)
- Configure firewall to limit SSH access
- Regular security updates

---

## ⚡ 13. Quick Reference (Commands)

### 🔹 Generate a local key
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### 🔹 Initial login (AWS EC2)
```bash
ssh -i ~/Downloads/your-aws-key.pem ubuntu@<SERVER_IP>
```

### 🔹 Basic SSH login
```bash
ssh username@<SERVER_IP>
ssh -i ~/.ssh/id_ed25519 ubuntu@<SERVER_IP>
```

### 🔹 Prepare `.ssh` folder
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

### 🔹 Add public key manually
```bash
echo "ssh-ed25519 AAAA... user@machine" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 🔹 CI/CD User Setup (Recommended Method)
```bash
sudo useradd -m -s /bin/bash emma
sudo usermod -aG sudo emma
sudo rsync --archive --chown=emma:emma ~/.ssh /home/emma/
sudo chmod 700 /home/emma/.ssh
sudo chmod 600 /home/emma/.ssh/authorized_keys
echo "emma ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/emma
sudo chmod 440 /etc/sudoers.d/emma
```

### 🔹 SSH Config (optional)
```bash
# ~/.ssh/config
Host myserver
    HostName <SERVER_IP>
    User emma
    IdentityFile ~/.ssh/id_ed25519
```

Connect with:
```bash
ssh myserver
```

---

## 🚨 14. Troubleshooting Common Issues

### ❌ "Permission denied (publickey)"
- Check `authorized_keys` permissions (must be 600)
- Verify public key is correctly copied
- Check SSH folder ownership

### ❌ "sudo: no tty present and no askpass program specified"
- Passwordless sudo not configured
- Check sudoers file syntax and permissions

### ❌ "Could not open a connection to your authentication agent"
- Run `ssh-add ~/.ssh/id_ed25519`

### ❌ CI/CD pipeline can't connect
- Verify private key format (should start with `-----BEGIN OPENSSH PRIVATE KEY-----`)
- Check firewall rules
- Verify username and IP address in secrets

### ❌ "sudo: parse error in /etc/sudoers.d/emma near line X"
- Syntax error in sudoers file
- Use `visudo -c` to check syntax
- Use `visudo -f /etc/sudoers.d/emma` to fix safely

### ❌ "sudo: /etc/sudoers.d/emma is world writable"
- Fix permissions: `sudo chmod 440 /etc/sudoers.d/emma`

---

## 📋 15. Complete Example Flow

```bash
# 1️⃣ Generate key locally
ssh-keygen -t ed25519 -C "ci-cd@company.com"

# 2️⃣ Login with .pem (first time)
ssh -i ~/Downloads/foudawire-key.pem ubuntu@13.218.36.173

# 3️⃣ Add key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=~/Downloads/foudawire-key.pem" ubuntu@13.218.36.173

# 4️⃣ Create CI/CD automation user
sudo useradd -m -s /bin/bash emma
sudo usermod -aG sudo emma
sudo rsync --archive --chown=emma:emma ~/.ssh /home/emma/
sudo chmod 700 /home/emma/.ssh
sudo chmod 600 /home/emma/.ssh/authorized_keys

# 5️⃣ Configure passwordless sudo (SAFER method)
echo "emma ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/emma
sudo chmod 440 /etc/sudoers.d/emma

# 6️⃣ Test automation access
ssh emma@13.218.36.173 "sudo whoami"

# 7️⃣ Secure SSH (optional)
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

---

## 🎯 Summary

This guide provides everything needed for:
- ✅ **Secure SSH access** to Ubuntu servers
- ✅ **CI/CD automation user** setup with passwordless sudo
- ✅ **Proper sudo configuration** using `/etc/sudoers.d/` (recommended)
- ✅ **Proper permissions** and security configurations
- ✅ **Troubleshooting** common issues
- ✅ **Best practices** for production environments

Your CI/CD pipelines can now securely deploy, install, and manage services automatically using the configured automation user.

---
