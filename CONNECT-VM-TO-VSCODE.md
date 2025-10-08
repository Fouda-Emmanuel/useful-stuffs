## VS Code Remote-SSH Setup for Any Linux VM (with Vagrant Appendix)

This guide explains how to connect **VS Code on Windows** (or any host OS) to a **Linux VM** (Ubuntu, CentOS, Debian, etc.) using **Remote-SSH**, including a dedicated section for **Vagrant users**.

---

## 1. Prerequisites

- Host machine: Windows, macOS, or Linux  
- Linux VM running Ubuntu, CentOS, Debian, or similar  
- SSH access to the VM (password or private key)  
- VS Code installed  
- VS Code Extensions:
  - **Remote - SSH**  
- Optional but recommended: Git installed on host  

---

## 2. Start / Access Your VM

### Generic VM

```bash
# Make sure your VM is running
# Identify VM IP, username, and authentication method

# Test SSH connection
ssh username@VM_IP

# Or with private key
ssh -i /path/to/private_key username@VM_IP

# You should see a Linux shell prompt
exit
````

### If using Vagrant

```bash
cd /path/to/vagrant/folder
vagrant status      # Check VM status
vagrant up          # Start VM if not running
vagrant ssh-config  # Get SSH connection details
```

---

## 3. Prepare SSH Configuration

Locate your SSH private key (if used).

Optional: add your VM to SSH config:

**Linux/macOS:** `~/.ssh/config`
**Windows:** `C:\Users\<User>\.ssh\config`

```text
Host my-vm
  HostName 192.168.56.101
  User vagrant
  Port 22
  IdentityFile "C:/path/to/private_key"  # skip if using password
  IdentitiesOnly yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

Replace `HostName`, `User`, `Port`, and `IdentityFile` according to your VM.

---

## 4. Connect VS Code to VM

1. Open VS Code on your host machine.
2. Press `Ctrl+Shift+P` → `Remote-SSH: Add New SSH Host`
3. Paste one of the following:

```bash
# Full SSH command
ssh -i "C:/path/to/private_key" username@VM_IP -p 22

# Or use SSH config alias
ssh my-vm
```

4. Save to default SSH config file
5. Connect: `Ctrl+Shift+P → Remote-SSH: Connect to Host... → select your host`
6. Wait for VS Code Server installation inside the VM (first-time only)

---

## 5. Open Your Project Folder

Options for folder locations inside VM:

* `/home/username/project` — project resides inside VM
* `/vagrant` — maps to host folder (optional)

`File → Open Folder → select folder`

You can now edit files as if they were local.

---

## 6. Using the Remote Terminal

Open terminal inside VS Code (`Terminal → New Terminal`). Commands run inside VM:

```bash
# Check versions
python3 --version
git status

# Navigate to project
cd /home/username/project

# Or clone repository
git clone https://github.com/<username>/<repo>.git
```

---

## 7. Disconnect / Reconnect

* `exit` → closes only the terminal, not VS Code connection
* Fully disconnect:

  * Click green Remote-SSH button → `Close Remote Connection`
  * Or `Ctrl+Shift+P → Remote-SSH: Close Remote Connection`
* Reconnecting later will reuse existing VS Code Server installation
* If the server folder is missing or VM was rebuilt, it will install again automatically

---

## 8. Troubleshooting

| Problem                          | Solution                                                                  |
| -------------------------------- | ------------------------------------------------------------------------- |
| Cannot establish connection      | Check VM is running, SSH IP/port correct                                  |
| Permission denied / key errors   | Ensure correct path to private key and permissions (`chmod 600` on Linux) |
| Multiple VS Code server versions | Remove old versions: `rm -rf ~/.vscode-server/cli/servers/Stable-*`       |
| Firewall blocks SSH              | Open port 22 (or custom) on VM and host firewall                          |
| VS Code Server download fails    | Check VM internet access; manually download server if needed              |

---

## 9. Best Practices

* Always use SSH keys over passwords for security
* Keep VM running while editing via VS Code
* Use synced/shared folders for convenience if host and VM need same files
* Backup `~/.vscode-server` in case of server corruption
* Name your hosts clearly in SSH config for multiple VMs

---

## 10. Workflow Diagram

```text
+------------------+       SSH       +----------------------+
|  VS Code (Host)  |  <---------->  |  Linux VM (Ubuntu,   |
|                  |                |  CentOS, Debian)     |
|  Remote-SSH Ext  |                |                      |
+------------------+                +----------------------+
         |                                  |
         | Edit / Save Files                | Files stored here
         |                                  |
         v                                  v
+------------------+                +----------------------+
| Host Folder / VM |  <---------->  | Project Folder in VM |
| /vagrant (optional)|              | /home/username/proj  |
+------------------+                +----------------------+
```

---

## 11. Vagrant-Specific Appendix

### 11.1 Start VM

```bash
cd /path/to/Vagrantfile
vagrant status      # Check VM status
vagrant up          # Start VM if not running
```

### 11.2 Get SSH Configuration

```bash
vagrant ssh-config
```

Example output:

```text
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  IdentityFile "C:/path/to/project/.vagrant/machines/default/virtualbox/private_key"
  IdentitiesOnly yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  PubkeyAcceptedKeyTypes +ssh-rsa
  HostKeyAlgorithms +ssh-rsa
```

### 11.3 Add SSH Host in VS Code

```bash
ssh -i "C:/path/to/project/.vagrant/machines/default/virtualbox/private_key" vagrant@127.0.0.1 -p 2222
```

Save in default SSH config and connect via Remote-SSH → VS Code will install server automatically.

### 11.4 Project Folder Options

* `/vagrant` → maps to host folder containing Vagrantfile
* `/home/vagrant/project` → VM-internal project folder

### 11.5 Common Vagrant Tips

* If VS Code asks to install server repeatedly, check `~/.vscode-server/bin` and remove old versions
* Always `vagrant up` before connecting
* Use `vagrant suspend` or `vagrant halt` to stop VM safely

---

## 12. Summary

1. Start your VM (Vagrant or other)
2. Test SSH connection
3. Add VM as SSH host in VS Code
4. Connect → allow VS Code Server installation
5. Open project folder → edit and run commands inside VM
6. Disconnect/reconnect safely when needed

---





