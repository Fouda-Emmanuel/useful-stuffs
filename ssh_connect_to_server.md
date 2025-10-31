# 🔐 Secure SSH Setup & Access Management Guide

This document provides a complete, step-by-step guide for setting up **secure SSH access** to an Ubuntu server (such as AWS EC2).
It covers generating keys, connecting to the server, creating a sudo user, configuring permissions, and applying best practices for security.

---

## 🧩 1. Generate a Local SSH Key (if you don’t have one)

Run this on your **local machine** (Linux/Mac/WSL):

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

* Press **Enter** to save to the default location:
  `~/.ssh/id_ed25519`
* Optionally set a **passphrase** for extra protection.

**Files created:**

| File                    | Description                                      |
| ----------------------- | ------------------------------------------------ |
| `~/.ssh/id_ed25519`     | 🔒 Private key (keep secret, never share)        |
| `~/.ssh/id_ed25519.pub` | 🔓 Public key (safe to share or copy to servers) |

---

## 🌍 2. Initial Login to Server (EC2 / Cloud)

If it’s your **first time connecting**, use the `.pem` key provided by your cloud host (e.g., AWS):

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

If it doesn’t exist, create it (after connecting with your `.pem` key):

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

If successful, you’ll log in without using the `.pem` file.

---

## 👤 7. Create a New Sudo User (Recommended)

It’s best practice to use a **dedicated user** for daily operations.

While logged in as `ubuntu`:

```bash
sudo useradd -m emma
sudo usermod -aG sudo emma
```

Then copy SSH access:

```bash
sudo rsync --archive --chown=emma:emma ~/.ssh /home/emma/
```

Now test login as the new user from your local machine:

```bash
ssh emma@<SERVER_IP>
```

✅ You now have your own sudo user with SSH access.

---

## ⚙️ 8. Optional: SSH Config for Convenience

Create or edit your local SSH config file:

```bash
nano ~/.ssh/config
```

Add this block:

```bash
Host foudawire
    HostName <SERVER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

Then connect easily using:

```bash
ssh foudawire
```

No need to type IP or key path anymore.

---

## 🔒 9. Optional: Disable Root & Password Login

After confirming your new user works perfectly, secure SSH further:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set:

```
PermitRootLogin no
PasswordAuthentication no
```

Then restart the SSH service:

```bash
sudo systemctl restart ssh
```

---

## 🧠 10. Best Practices

✅ **Never share** your private key
✅ Keep your `.pem` file safe and read-only (`chmod 400`)
✅ Use **one key per user or device**
✅ Rotate keys regularly (update `~/.ssh/authorized_keys`)
✅ Use `sudo` for admin tasks — never log in directly as root
✅ Make backups of your `.ssh` folder

---

## ⚡ 11. Quick Reference (Commands)

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

### 🔹 SSH Config (optional)

```bash
# ~/.ssh/config
Host foudawire
    HostName <SERVER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

Connect with:

```bash
ssh foudawire
```

### 🔹 Create and configure new user

```bash
sudo useradd -m emma
sudo usermod -aG sudo emma
sudo rsync --archive --chown=emma:emma ~/.ssh /home/emma/
ssh emma@<SERVER_IP>
```

### 🔹 Root or sudo usage

```bash
sudo -i
sudo <command>
```

---

### 🧾 Example Summary Flow

```bash
# 1️⃣ Generate key
ssh-keygen -t ed25519 -C "you@example.com"

# 2️⃣ Login with .pem (first time)
ssh -i ~/Downloads/foudawire-key.pem ubuntu@13.218.36.173

# 3️⃣ Add key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=~/Downloads/foudawire-key.pem" ubuntu@13.218.36.173

# 4️⃣ Create new sudo user
sudo useradd -m emma
sudo usermod -aG sudo emma
sudo rsync --archive --chown=emma:emma ~/.ssh /home/emma/

# 5️⃣ Login as new user
ssh emma@13.218.36.173
```

---

