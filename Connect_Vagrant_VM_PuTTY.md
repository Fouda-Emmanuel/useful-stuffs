# Connecting to Local Vagrant Lab VMs Using PuTTY

## Overview

This guide explains how to connect to **local Vagrant virtual machines (VMs)** using **PuTTY** on Windows. It mirrors a real-world AWS EC2 setup but runs entirely on your local machine, allowing safe experimentation without cloud costs.

You will learn how to:

* Start your Vagrant lab environment
* Check SSH configuration
* Generate PuTTY-compatible private keys
* Connect to each VM via PuTTY
* Understand VM networking and private/public access

 **Why use PuTTY with Vagrant?**  

 Vagrant creates Linux VMs with OpenSSH keys. Windows users need PuTTY to SSH, which requires converting these keys into `.ppk` format. This guide ensures **passwordless and secure connections** to your local VMs.

---

## Prerequisites

1. **Installed software**:

   * [Vagrant](https://www.vagrantup.com/downloads)
   * [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
   * [PuTTY](https://www.putty.org/)
   * [PuTTYgen](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

2. **A prepared Vagrant project** with at least 4 VMs (Ubuntu + CentOS):

```
ansible-lab/
├── Vagrantfile
├── control.ppk
├── web01.ppk
├── web02.ppk
└── db01.ppk
```

> **Note:** The `.ppk` files will be created during Step 3.

---

## Step 1: Start Your Vagrant Lab

Open a terminal (PowerShell, Git Bash, or CMD):

```bash
cd path/to/ansible-lab
vagrant up
```

* This will **create and start all VMs**.
* To check which VMs are running:

```bash
vagrant status
```

> The VMs will have both **private network IPs** for inter-VM communication and **public network ports** for SSH from your laptop.

---

## Step 2: Check SSH Configuration

Vagrant automatically generates SSH connection info. Run:

```bash
vagrant ssh-config
```

Example output for one VM:

```
Host control
  HostName 127.0.0.1
  User vagrant
  Port 2222
  IdentityFile "C:/path/to/ansible-lab/.vagrant/machines/control/virtualbox/private_key"
```

> **Explanation:**
>
> * **HostName** → IP for SSH (`127.0.0.1` is localhost)
> * **Port** → forwarded port (unique per VM)
> * **IdentityFile** → path to the private key used for passwordless login

---

## Step 3: Generate PuTTY Private Keys (.ppk)

PuTTY cannot use OpenSSH private keys directly. Convert them:

1. Open **PuTTYgen**.
2. Click **Load** → select the Vagrant private key:

```
<project-folder>/.vagrant/machines/<VM_NAME>/virtualbox/private_key
```

3. Click **Save private key** → choose a name like `control.ppk` in your project folder.
4. Click **Yes** if it warns about saving without a passphrase.

> Repeat for each VM (`control`, `web01`, `web02`, `db01`).

---

## Step 4: Configure PuTTY Sessions

1. Open **PuTTY**.
2. **Session → Host Name:** `127.0.0.1`
3. **Port:** Use the port from `vagrant ssh-config` (see table below)
4. **Connection type:** SSH
5. **Connection → Data → Auto-login username:** `vagrant`
6. **Connection → SSH → Auth → Private key file:** select corresponding `.ppk`
7. Back to **Session → Saved Sessions:** give a name (e.g., `control`) → click **Save**
8. Click **Open** → you should log in directly to the VM shell.

---

## Step 5: VM Connection Details

| VM      | SSH Port | Private Key File | PuTTY Session Name |
| ------- | -------- | ---------------- | ------------------ |
| control | 2222     | control.ppk      | control            |
| web01   | 2200     | web01.ppk        | web01              |
| web02   | 2201     | web02.ppk        | web02              |
| db01    | 2202     | db01.ppk         | db01               |

> **Tip:** Save these sessions in PuTTY for easy access.

---

## Step 6: Optional – Test Inter-VM Connectivity

From the control VM:

```bash
vagrant ssh control
```

Ping the other VMs (using **private network IPs**):

```bash
ping 192.168.56.11  # web01
ping 192.168.56.12  # web02
ping 192.168.56.13  # db01
```

> This simulates **AWS private network** behavior — VMs can communicate internally, but external SSH goes through the control node.

---

## Step 7: Network Diagram

```
       ┌─────────┐
       │ Laptop  │
       │ PuTTY   │
       └────┬────┘
            │ SSH (127.0.0.1 + forwarded port)
            ▼
       ┌───────────┐
       │ control   │
       │ Ubuntu VM │
       └───┬───────┘
           │ Private Network
           ▼
   ┌────────┬────────┬────────┐
   │ web01  │ web02  │ db01   │
   │ CentOS │ CentOS │ CentOS │
   └────────┴────────┴────────┘
```

* **Laptop → control**: public network SSH via forwarded port
* **control → other VMs**: private network, internal communication

---

## Step 8: Quick Start Reference

For fast connection:

| VM      | Host      | Port | User    | Key File    |
| ------- | --------- | ---- | ------- | ----------- |
| control | 127.0.0.1 | 2222 | vagrant | control.ppk |
| web01   | 127.0.0.1 | 2200 | vagrant | web01.ppk   |
| web02   | 127.0.0.1 | 2201 | vagrant | web02.ppk   |
| db01    | 127.0.0.1 | 2202 | vagrant | db01.ppk    |

1. Open PuTTY → select session → Open
2. You are directly in the VM shell

---

## Tips & Best Practices

* Always `vagrant up` before connecting with PuTTY.
* Save sessions in PuTTY for quick access.
* `.ppk` files are sensitive — **do not share publicly**.
* Use private network IPs for inter-VM communication.
* Use public network only for direct SSH access from your laptop.

---

### Summary

This guide provides a complete workflow for connecting to local Vagrant VMs using PuTTY on Windows. The methodology replicates enterprise AWS EC2 SSH access patterns in a local development environment, enabling realistic infrastructure testing and configuration management practice.
