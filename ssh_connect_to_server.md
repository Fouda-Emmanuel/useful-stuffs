# **SSH Key-Based Login Guide (Linux / EC2)**

This guide explains how to securely connect to a Linux server using **SSH key authentication**, including generating keys, copying them to the server, and logging in without passwords.

---

## **1. Generate a local SSH key (if you don’t have one)**

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

* Default location: `~/.ssh/id_ed25519`
* Optional passphrase for extra security
* Files created:

  * Private key: `~/.ssh/id_ed25519` (**keep secret**)
  * Public key: `~/.ssh/id_ed25519.pub` (**safe to share**)

---

## **2. Initial login to server**

### **EC2 / Cloud servers**

* AWS EC2 instances provide a `.pem` key at launch
* Default user for Ubuntu EC2: `ubuntu`

```bash
ssh -i ~/Downloads/your-aws-key.pem ubuntu@<SERVER_IP>
```

* Replace `<SERVER_IP>` with your server’s IP

> **Note:** Direct `root` login is disabled for security. Use `ubuntu` and `sudo` for root commands.

---

## **3. Basic SSH command**

Once your key is added, you can connect using the simple form:

```bash
ssh username@ip
```

* `username` → the server user (e.g., `ubuntu`)
* `ip` → server IP or public DNS

Example with your local key:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@13.218.36.173
```

---

## **4. Prepare `.ssh` folder on the server**

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

* Ensures the folder exists and is secure

---

## **5. Add your local public key to the server**

### **Option 1: Manual copy**

1. On your local machine:

```bash
cat ~/.ssh/id_ed25519.pub
```

* Copy the full line

2. On the server (while logged in with `.pem`):

```bash
echo "PASTE_YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

* Appends your key **without removing the existing `.pem` key**

---

### **Option 2: Automatic copy with `ssh-copy-id`**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=~/Downloads/your-aws-key.pem" ubuntu@<SERVER_IP>
```

* Logs in with the `.pem` key and adds your local key automatically

---

## **6. Test passwordless login**

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<SERVER_IP>
```

* You should log in **without using the `.pem` file**

---

## **7. Optional: SSH config for convenience**

Create or edit `~/.ssh/config` on your local machine:

```text
Host foudawire
    HostName <SERVER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

* Then you can simply run:

```bash
ssh foudawire
```

* No need to type `-i` or the full IP

---

## **8. Using root privileges**

* Use `sudo` when logged in as `ubuntu`:

```bash
sudo -i       # switch to root
sudo <command> # run a command as root
```

> Do **not** enable root SSH login on AWS for security reasons

---

## **9. Best Practices**

* Never share your private key
* Permissions:

  * `.ssh` folder → `700`
  * `authorized_keys` → `600`
* Use separate keys for each device or user
* Keep your `.pem` file secure (for EC2 initial login)

---

## **10. Quick Reference (Commands)**

```bash
# Generate local key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Initial login to EC2
ssh -i ~/Downloads/your-aws-key.pem ubuntu@<SERVER_IP>

# Basic SSH login
ssh username@ip
ssh -i ~/.ssh/id_ed25519 ubuntu@<SERVER_IP>

# Create .ssh folder
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Add your public key manually
echo "ssh-ed25519 AAAA... user@machine" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Optional: SSH config
# ~/.ssh/config
Host foudawire
    HostName <SERVER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

# connect
ssh username@ip

# Connect via shortcut
ssh foudawire

# Use sudo for root
sudo -i
```

---

