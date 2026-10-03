# Kubernetes Node Setup — containerd + kubeadm/kubelet

A reproducible, version-pinned installation guide for preparing an Ubuntu 24.04 (Noble) node to join or initialize a Kubernetes cluster.

- **OS:** Ubuntu 24.04 LTS (Noble Numbat), amd64
- **Container runtime:** `containerd.io` **2.2.6**
- **Kubernetes:** **v1.35** (`kubeadm` + `kubelet` **1.35.8-1.1**)
- **Cgroup driver:** `systemd`

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Install & Configure containerd](#step-1--install--configure-containerd)
4. [Step 2 — Install kubeadm & kubelet](#step-2--install-kubeadm--kubelet)
5. [Verification](#verification)
6. [What This Setup Does NOT Do](#what-this-setup-does-not-do)
7. [Next Steps](#next-steps)
8. [Notes & Caveats](#notes--caveats)

---

## Overview

This repository documents the exact steps required to turn a fresh Ubuntu 24.04 machine into a Kubernetes-ready node. Two scripts are provided:

| Script | Purpose |
|--------|---------|
| `install-containerd.sh` | Installs and configures containerd 2.2.6 as the Kubernetes CRI runtime |
| `install-kube-tools.sh` | Installs `kubeadm` and `kubelet` at a pinned Kubernetes version |

Both scripts use **version pinning** and **`apt-mark hold`** so a cluster does not accidentally break from an unattended upgrade.

---

## Prerequisites

- Ubuntu 24.04 (Noble) on `amd64`
- `sudo` privileges
- Internet access to `download.docker.com` and `pkgs.k8s.io`
- (Recommended for cluster use)
  - Swap disabled
  - Unique hostname, MAC address, and `product_uuid` per node
  - Required ports open (see [Next Steps](#next-steps))

---

## Step 1 — Install & Configure containerd

### Script: `install-containerd.sh`

```bash
#!/bin/bash
set -e

echo "=== 1. Installing repository prerequisites ==="
sudo apt-get update
sudo apt-get install -y ca-certificates curl

echo "=== 2. Installing Docker repository GPG key ==="
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "=== 3. Adding Docker APT repository ==="
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

echo "=== 4. Updating package lists ==="
sudo apt-get update

echo "=== 5. Installing containerd.io 2.2.6 ==="
sudo apt-get install -y \
  containerd.io=2.2.6-1~ubuntu.24.04~noble

echo "=== 6. Preventing automatic containerd upgrade ==="
sudo apt-mark hold containerd.io

echo "=== 7. Generating containerd configuration ==="
sudo mkdir -p /etc/containerd

containerd config default \
  | sudo tee /etc/containerd/config.toml > /dev/null

echo "=== 8. Configuring systemd cgroups ==="
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/g' \
  /etc/containerd/config.toml

echo "=== 9. Configuring Kubernetes sandbox image ==="
sudo sed -i \
  's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.10"#' \
  /etc/containerd/config.toml

echo "=== 10. Starting containerd ==="
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

echo "=== 11. Verification ==="

echo "--- containerd version ---"
containerd --version

echo "--- containerd package ---"
dpkg -l | grep containerd.io

echo "--- systemd cgroup ---"
grep 'SystemdCgroup' /etc/containerd/config.toml

echo "--- sandbox image ---"
grep -i pause /etc/containerd/config.toml

echo "--- service status ---"
systemctl is-active containerd

echo "=== containerd installation complete ==="
```

### Step-by-Step Breakdown

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Install `ca-certificates`, `curl` | Prerequisites for fetching GPG keys over HTTPS |
| 2 | Install Docker GPG key to `/etc/apt/keyrings/docker.asc` | Verifies authenticity of Docker packages |
| 3 | Add Docker APT repo (`noble stable`) | Source for the `containerd.io` package |
| 4 | `apt-get update` | Refresh package index with the new repo |
| 5 | Install pinned `containerd.io=2.2.6-1~ubuntu.24.04~noble` | Reproducible runtime version |
| 6 | `apt-mark hold containerd.io` | Prevent unintended upgrades |
| 7 | Generate default config | Baseline `/etc/containerd/config.toml` |
| 8 | Set `SystemdCgroup = true` | Match kubelet's `systemd` cgroup driver |
| 9 | Set `sandbox_image` to `pause:3.10` | Kubernetes 1.30+ default pause image |
| 10 | Enable + start `containerd` | Activate the service |
| 11 | Verify version, hold status, config, service | Confirm successful deployment |

### Key Configuration Choices

- **`SystemdCgroup = true`** — Kubernetes kubelet defaults to the `systemd` cgroup driver; containerd must match or pods will fail to start.
- **`sandbox_image = registry.k8s.io/pause:3.10`** — The pause image used for pod sandboxes (required for Kubernetes ≥ 1.30).
- **`apt-mark hold`** — Locks containerd at 2.2.6, avoiding accidental upgrades that could break a running cluster.

---

## Step 2 — Install kubeadm & kubelet

### Script: `install-kube-tools.sh`

```bash
#!/bin/bash
set -e

K8S_VERSION="1.35"
K8S_PACKAGE_VERSION="1.35.8-1.1"

echo "=========================================="
echo " Kubernetes ${K8S_PACKAGE_VERSION}"
echo " Installing kubeadm + kubelet"
echo "=========================================="

# --------------------------------------------------
# 1. Remove any old Kubernetes repository/key
# --------------------------------------------------

sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# --------------------------------------------------
# 2. Prerequisites
# --------------------------------------------------

sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg

# --------------------------------------------------
# 3. Kubernetes repository key
# --------------------------------------------------

sudo mkdir -p -m 0755 /etc/apt/keyrings

curl -fsSL \
  https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# --------------------------------------------------
# 4. Kubernetes v1.35 repository
# --------------------------------------------------

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# --------------------------------------------------
# 5. Install EXACT Kubernetes version
# --------------------------------------------------

sudo apt-get install -y \
  kubelet=${K8S_PACKAGE_VERSION} \
  kubeadm=${K8S_PACKAGE_VERSION}

# --------------------------------------------------
# 6. Prevent automatic package upgrades
# --------------------------------------------------

sudo apt-mark hold kubelet kubeadm

# --------------------------------------------------
# 7. Enable kubelet
#    Do NOT start it manually yet.
# --------------------------------------------------

sudo systemctl enable kubelet

# --------------------------------------------------
# 8. Verification
# --------------------------------------------------

echo
echo "========== Kubernetes versions =========="

kubeadm version
kubelet --version

echo
echo "========== Held packages =========="

apt-mark showhold

echo
echo "========== Kubelet service =========="

systemctl is-enabled kubelet
systemctl is-active kubelet || true

echo
echo "=========================================="
echo " Kubernetes installation completed."
echo " kubelet is ENABLED but intentionally NOT"
echo " started manually before kubeadm init/join."
echo "=========================================="
```

### Step-by-Step Breakdown

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Remove old `kubernetes.list` and keyring | Clean slate, avoid conflicting repos |
| 2 | Install `ca-certificates`, `curl`, `gpg` | Prerequisites for fetching the signing key |
| 3 | Fetch & dearmor the Kubernetes v1.35 `Release.key` | Verifies authenticity of K8s packages |
| 4 | Add the v1.35 `pkgs.k8s.io` repo | Source for `kubeadm` and `kubelet` |
| 5 | Install **exact** `1.35.8-1.1` | Reproducible control-plane/tooling version |
| 6 | `apt-mark hold kubelet kubeadm` | Prevent auto-upgrades from breaking the cluster |
| 7 | `systemctl enable kubelet` | Ensure it starts on boot — **do not start manually** |
| 8 | Print versions, held packages, service state | Confirm the install |

### Why kubelet is Enabled but NOT Started

`kubelet` will crash-loop until it is given either:

- a valid `kubeadm init` configuration (control plane node), or
- a valid `kubeadm join` token (worker node).

Starting it manually before those steps only produces confusing error logs. `kubeadm init` / `kubeadm join` will start it correctly.

---

## Verification

After running both scripts, you should see output similar to:

```text
--- containerd version ---
containerd github.com/containerd/containerd v2.2.6 ...

--- containerd package ---
ii  containerd.io  2.2.6-1~ubuntu.24.04~noble  amd64

--- systemd cgroup ---
SystemdCgroup = true

--- sandbox image ---
sandbox_image = "registry.k8s.io/pause:3.10"

--- service status ---
active

--- kubeadm version ---
kubeadm version: &version.Info{Major:"1", Minor:"35", ...}

--- kubelet version ---
Kubernetes v1.35.8

--- held packages ---
containerd.io
kubeadm
kubelet

--- kubelet service ---
enabled
inactive   # expected before kubeadm init/join
```

### Manual Verification Commands

```bash
# containerd
containerd --version
dpkg -l | grep containerd.io
grep SystemdCgroup /etc/containerd/config.toml
grep -i pause     /etc/containerd/config.toml
systemctl is-active containerd

# Kubernetes tools
kubeadm version
kubelet --version
apt-mark showhold
systemctl is-enabled kubelet
```

---

## What This Setup Does NOT Do

These items are intentionally **out of scope** and must be handled separately:

- **CNI plugin** — install a pod network (Calico, Flannel, Cilium, etc.) after `kubeadm init`.
- **Swap** — Kubernetes requires swap off:
  ```bash
  sudo swapoff -a
  sudo sed -i '/ swap / s/^/#/' /etc/fstab
  ```
- **Kernel modules & sysctl** — for `br_netfilter`, `overlay`, and IP forwarding.
- **Container runtime socket for kubelet** — `kubeadm` autodetects `unix:///run/containerd/containerd.sock`; if using a custom config, set it explicitly.
- **Firewall rules** — open the required control-plane / worker ports.
- **Cluster initialization** — `kubeadm init` / `kubeadm join` are manual follow-ups.
- **kubectl** — not installed by these scripts. Install separately if needed:
  ```bash
  sudo apt-get install -y kubectl=1.35.8-1.1
  sudo apt-mark hold kubectl
  ```

---

## Next Steps

After both scripts succeed:

1. **Disable swap** (if not already):
   ```bash
   sudo swapoff -a
   sudo sed -i '/ swap / s/^/#/' /etc/fstab
   ```

2. **Load kernel modules**:
   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```

3. **Set sysctl parameters**:
   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   EOF

   sudo sysctl --system
   ```

4. **Initialize the control plane** (first node only):
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   ```

5. **Join additional nodes** using the token printed by `kubeadm init`.

---

## Notes & Caveats

- **Architecture is hardcoded to `amd64`.** For arm64, change the containerd repo line to `arch=arm64`.
- **Ubuntu 24.04 / Noble only.** Different Ubuntu releases require a different repo suite (`jammy`, `focal`, etc.).
- **`apt-mark hold` is intentional.** To upgrade later, `apt-mark unhold`, install the new pin, and re-hold.
- **Idempotency.** The Kubernetes script wipes and re-adds its repo each run. The containerd script is not fully idempotent (`apt-mark hold` will simply report "already held").
- **`set -e`** is enabled in both scripts — any failing command aborts the script. Read the output carefully if something stops early.
- **Matching versions matters.** `kubeadm`, `kubelet`, and (if installed) `kubectl` should all be the **same minor version** (here `1.35.x`), and never more than one minor ahead/behind the control plane.
- **Do not start `kubelet` manually** before `kubeadm init` or `kubeadm join`. It is enabled for boot, but `kubeadm` orchestrates its first real start.

---
