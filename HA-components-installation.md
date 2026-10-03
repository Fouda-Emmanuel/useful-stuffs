# Kubernetes Node Setup — containerd + kubeadm/kubelet + crictl + kubectl (Bastion)

A reproducible, version-pinned installation guide for preparing:

- **Ubuntu 24.04 (Noble) Kubernetes nodes** — containerd, crictl, kubelet, kubeadm
- **An Ubuntu 24.04 bastion host** — kubectl only

- **OS:** Ubuntu 24.04 LTS (Noble Numbat), amd64
- **Container runtime:** `containerd.io` **2.2.6**
- **cri-tools:** `crictl` **v1.35.0**
- **Kubernetes:** **v1.35** (`kubeadm` + `kubelet` + `kubectl` **1.35.8-1.1**)
- **Cgroup driver:** `systemd`

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Role Matrix](#role-matrix)
4. [Prerequisites](#prerequisites)
5. [Checking Available Repository Versions](#checking-available-repository-versions)
   - [containerd.io versions](#containerdio-versions)
   - [kubeadm / kubelet / kubectl versions](#kubeadm--kubelet--kubectl-versions)
   - [crictl versions](#crictl-versions)
6. [Step 1 — Install & Configure containerd (nodes only)](#step-1--install--configure-containerd-nodes-only)
7. [Step 2 — Install crictl (nodes only)](#step-2--install-crictl-nodes-only)
8. [Step 3 — Verify CRI is Exposed by containerd (nodes only)](#step-3--verify-cri-is-exposed-by-containerd-nodes-only)
9. [Step 4 — Install kubeadm & kubelet (nodes only)](#step-4--install-kubeadm--kubelet-nodes-only)
10. [Step 5 — Install kubectl (bastion only)](#step-5--install-kubectl-bastion-only)
11. [Verification](#verification)
12. [What This Setup Does NOT Do](#what-this-setup-does-not-do)
13. [Next Steps](#next-steps)
14. [Troubleshooting: `crictl info` fails or `grpc.v1.cri` missing](#troubleshooting-crictl-info-fails-or-grpcv1cri-missing)
15. [Notes & Caveats](#notes--caveats)

---

## Overview

This repository documents the exact steps required to turn a fresh Ubuntu 24.04 machine into either:

- a **Kubernetes node** (control plane or worker), or
- a **bastion / admin host** that only talks to the cluster API.

Four scripts are provided:

| Script | Target | Purpose |
|--------|--------|---------|
| `install-containerd.sh` | K8s nodes | Installs and configures containerd 2.2.6 as the CRI runtime |
| `install-crictl.sh`     | K8s nodes | Installs `crictl` v1.35.0 and points it at containerd |
| `install-kube-tools.sh` | K8s nodes | Installs `kubeadm` and `kubelet` at a pinned version |
| `install-kubectl.sh`    | Bastion   | Installs `kubectl` only at a pinned version |

All scripts use **version pinning** and **`apt-mark hold`** so a cluster does not accidentally break from an unattended upgrade.

---

## Architecture

```
                         AWS
                          │
                    ┌─────┴─────┐
                    │  Bastion   │
                    │            │
                    │ kubectl    │
                    └─────┬──────┘
                          │
                    kubectl / API
                          │
                ┌─────────▼─────────┐
                │    HAProxy/DNS    │
                │  controlPlane     │
                │    Endpoint       │
                └─────────┬─────────┘
                          │ :6443
             ┌────────────┼────────────┐
             ▼            ▼            ▼
           cp01         cp02         cp03
         kubeadm       kubeadm       kubeadm
         kubelet       kubelet       kubelet
         containerd    containerd    containerd
         crictl        crictl        crictl
             │            │            │
             └────────────┼────────────┘
                          │
                    ┌─────┴─────┐
                    ▼           ▼
                 worker1     worker2
```

The bastion is an **administrative access point**, not a Kubernetes node. It does not run `containerd`, `kubelet`, `kubeadm`, or `crictl` — only `kubectl`.

---

## Role Matrix

| Node    | containerd | crictl  | kubelet | kubeadm | kubectl |
|---------|:----------:|:-------:|:-------:|:-------:|:-------:|
| cp01    | ✅ 2.2.6   | 1.35.0  | 1.35.8  | 1.35.8  | —       |
| cp02    | ✅ 2.2.6   | 1.35.0  | 1.35.8  | 1.35.8  | —       |
| cp03    | ✅ 2.2.6   | 1.35.0  | 1.35.8  | 1.35.8  | —       |
| worker1 | ✅ 2.2.6   | 1.35.0  | 1.35.8  | 1.35.8  | —       |
| worker2 | ✅ 2.2.6   | 1.35.0  | 1.35.8  | 1.35.8  | —       |
| bastion | —          | —       | —       | —       | 1.35.8  |

---

## Prerequisites

- Ubuntu 24.04 (Noble) on `amd64`
- `sudo` privileges
- Internet access to `download.docker.com`, `pkgs.k8s.io`, and `github.com`
- (Recommended for cluster nodes)
  - Swap disabled
  - Unique hostname, MAC address, and `product_uuid` per node
  - Required ports open (see [Next Steps](#next-steps))

---

## Checking Available Repository Versions

Before installing **any** pinned component, it is good practice to inspect what versions the repository actually offers. Repositories get new patch releases over time, and pinning to a version that no longer exists (or that was superseded) will fail.

> The following commands are **read-only**. Run them **after** the repository has been configured (i.e. after Step 3 of the corresponding script, but **before** installing).

### containerd.io versions

**Repository:** `download.docker.com/linux/ubuntu` (`noble`, `stable`)

Configure the repo first (see [Step 2 in `install-containerd.sh`](#step-2--install--configure-containerd-nodes-only)), then:

```bash
# List every available containerd.io version, newest first
apt-cache madison containerd.io
```

Example output (changes over time):

```
containerd.io | 2.2.6-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
containerd.io | 2.2.5-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
containerd.io | 2.2.4-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
...
containerd.io | 2.1.0-1~ubuntu.24.04~noble | https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
```

Then check the current candidate (the version `apt` would install by default):

```bash
apt-cache policy containerd.io
```

```
containerd.io:
  Installed: (none)
  Candidate: 2.2.6-1~ubuntu.24.04~noble
  Version table:
     2.2.6-1~ubuntu.24.04~noble 500
        500 https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
     2.2.5-1~ubuntu.24.04~noble 500
        500 https://download.docker.com/linux/ubuntu noble/stable amd64 Packages
     ...
```

**What the version string means:**

```
2.2.6-1~ubuntu.24.04~noble
│ │ │  │       │     │
│ │ │  │       │     └── Ubuntu release (noble = 24.04)
│ │ │  │       └──────── Ubuntu version
│ │ │  └──────────────── Debian/package revision
│ │ └─────────────────── PATCH
│ └───────────────────── MINOR
└─────────────────────── MAJOR
```

- **MAJOR** — breaking changes (e.g. `2.x` vs `1.x`)
- **MINOR** — new features, backward compatible
- **PATCH** — bug/security fixes only

**Install a specific version:**

```bash
sudo apt-get install -y containerd.io=2.2.6-1~ubuntu.24.04~noble
```

**Check what is actually installed:**

```bash
# Package version
dpkg -l | grep containerd.io

# Binary version (from the daemon itself)
containerd --version
```

Example:

```
containerd github.com/containerd/containerd v2.2.6 1234567890abcdef...
```

---

### kubeadm / kubelet / kubectl versions

**Repository:** `pkgs.k8s.io/core:/stable:/v1.35/deb`

Configure the repo first (see [Step 4 in `install-kube-tools.sh`](#step-4--kubernetes-v135-repository) or the equivalent in `install-kubectl.sh`), then:

```bash
# List every available version of each component
apt-cache madison kubeadm
apt-cache madison kubelet
apt-cache madison kubectl
```

Or all at once:

```bash
apt-cache madison kubeadm kubelet kubectl
```

Example output (changes over time):

```
kubeadm | 1.35.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
kubeadm | 1.35.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
kubeadm | 1.35.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
...
kubeadm | 1.35.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages

kubelet | 1.35.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
kubelet | 1.35.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
...

kubectl | 1.35.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
kubectl | 1.35.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
...
```

Then check the current candidate for each:

```bash
apt-cache policy kubeadm
apt-cache policy kubelet
apt-cache policy kubectl
```

Example:

```
kubectl:
  Installed: (none)
  Candidate: 1.35.9-1.1
  Version table:
     1.35.9-1.1 500
        500 https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
     1.35.8-1.1 500
        500 https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
     ...
```

**What the version string means:**

```
1.35.8-1.1
│ │  │ │ │
│ │  │ │ └── Build/revision within the deb
│ │  │ └──── Debian revision
│ │  └────── PATCH
│ └───────── MINOR
└─────────── MAJOR
```

- **MAJOR.MINOR** — Kubernetes release line (e.g. `1.35`)
- **PATCH** — `1.35.8` is the 8th patch in the 1.35 line
- **`-1.1`** — Debian package revision

> **Kubernetes skew policy:** `kubelet` may be up to **one minor older** than `kube-apiserver`. `kubectl` may be **one minor newer or older**. Patch versions should match across components.

**Install specific versions:**

```bash
# Nodes
sudo apt-get install -y kubelet=1.35.8-1.1 kubeadm=1.35.8-1.1

# Bastion
sudo apt-get install -y kubectl=1.35.8-1.1
```

**Check what is actually installed:**

```bash
# Package versions
dpkg -l | grep -E 'kubeadm|kubelet|kubectl'

# Binary versions
kubeadm version
kubelet --version
kubectl version --client
```

Example:

```
kubeadm version: &version.Info{Major:"1", Minor:"35", GitVersion:"v1.35.8", ...}
Kubernetes v1.35.8
Client Version: v1.35.8
Kustomize Version: v5.x.x
```

> **Tip:** the `GitVersion` field is the authoritative upstream version (e.g. `v1.35.8`), while the deb version adds `-1.1`.

---

### crictl versions

**Source:** `github.com/kubernetes-sigs/cri-tools/releases`

Unlike containerd/kubeadm, `crictl` is **not distributed via APT** — it is a static Go binary shipped as a tarball on GitHub Releases. There is no `apt-cache madison` equivalent; instead, query the GitHub API.

**List every published `cri-tools` release (tag names):**

```bash
curl -fsSL https://api.github.com/repos/kubernetes-sigs/cri-tools/releases \
  | grep '"tag_name"' \
  | sed -E 's/.*"tag_name": *"([^"]+)".*/\1/'
```

Example output (changes over time):

```
v1.35.0
v1.34.0
v1.33.0
v1.32.0
...
v1.24.2
```

**Check whether a specific version exists** (useful before pinning):

```bash
CRICTL_VERSION="v1.35.0"
curl -fsSI \
  "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz" \
  | head -n 1
```

Expected:

```
HTTP/2 302
```

(A `302` redirect to the CDN means the artifact exists; a `404` means the version or arch does not.)

**What the version string means:**

```
v1.35.0
│ │  │
│ │  └── PATCH — cri-tools patch
│ └───── MINOR — matches the Kubernetes minor line (1.35)
└─────── MAJOR
```

> **Alignment rule:** `crictl` minor **matches the Kubernetes minor** (e.g. `crictl v1.35.x` for a `v1.35` cluster). Patch versions of `crictl` are decoupled from Kubernetes patch versions.

**Check what is actually installed:**

```bash
crictl --version
```

Example:

```
crictl version v1.35.0
```

**List what is available locally (mirror of the runtime):**

```bash
sudo crictl images
sudo crictl pods
```

---

### Why pin instead of using the latest?

```
Repository
    │
    ├── 1.35.9  ← newer available
    ├── 1.35.8  ← OUR chosen version
    ├── 1.35.7
    └── ...
              │
              ▼
       install EXACTLY
          1.35.8
              │
              ▼
            HOLD
```

Pinning a specific patch version keeps every node (and the bastion) in lockstep. Upgrades become an explicit, deliberate action rather than a side effect of `apt-get upgrade`.

---

## Step 1 — Install & Configure containerd (nodes only)

> **Applies to:** cp01, cp02, cp03, worker1, worker2
> **Not applicable to:** bastion

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

## Step 2 — Install crictl (nodes only)

> **Applies to:** cp01, cp02, cp03, worker1, worker2
> **Not applicable to:** bastion

`crictl` is a CLI for talking directly to the CRI socket. It is invaluable for debugging pod sandboxes, pulling images, and inspecting the runtime **without** going through the Kubernetes API. It is a static binary, so no APT repository is needed.

### Script: `install-crictl.sh`

```bash
#!/bin/bash
set -e

CRICTL_VERSION="v1.35.0"

echo "=========================================="
echo " Installing crictl ${CRICTL_VERSION}"
echo "=========================================="

# --------------------------------------------------
# 1. Download crictl
# --------------------------------------------------

curl -LO \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz

# --------------------------------------------------
# 2. Install crictl
# --------------------------------------------------

sudo tar zxvf \
  crictl-${CRICTL_VERSION}-linux-amd64.tar.gz \
  -C /usr/local/bin

# --------------------------------------------------
# 3. Remove downloaded archive
# --------------------------------------------------

rm -f crictl-${CRICTL_VERSION}-linux-amd64.tar.gz

# --------------------------------------------------
# 4. Configure crictl to use containerd
# --------------------------------------------------

sudo tee /etc/crictl.yaml > /dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# --------------------------------------------------
# 5. Verification
# --------------------------------------------------

echo
echo "========== crictl version =========="

crictl --version

echo
echo "========== crictl configuration =========="

sudo cat /etc/crictl.yaml

echo
echo "========== containerd CRI test =========="

sudo crictl info > /dev/null

echo "crictl can communicate with containerd successfully."

echo
echo "========== Images =========="

sudo crictl images

echo
echo "=========================================="
echo " crictl ${CRICTL_VERSION} installed."
echo "=========================================="
```

### Step-by-Step Breakdown

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Download `crictl-v1.35.0-linux-amd64.tar.gz` from GitHub Releases | Source for the static binary |
| 2 | Extract to `/usr/local/bin` | Places `crictl` on `$PATH` |
| 3 | Remove the tarball | Clean up |
| 4 | Write `/etc/crictl.yaml` pointing at containerd's socket | Tells `crictl` which runtime to talk to |
| 5 | Print version, config, run `crictl info`, list images | Confirm the install |

### Why `/etc/crictl.yaml`?

`crictl` supports many runtimes. Without a config it tries to auto-detect a socket — which is fragile in scripts. Pinning the endpoints makes the tool deterministic:

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint:   unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```

Both endpoints are the same because containerd exposes both `RuntimeService` and `ImageService` over the same gRPC socket.

---

## Step 3 — Verify CRI is Exposed by containerd (nodes only)

> **Applies to:** cp01, cp02, cp03, worker1, worker2
> **Do this BEFORE moving on to kubeadm/kubelet.**

### Why this step exists

In containerd **2.x**, the CRI plugin is split internally into:

- `io.containerd.cri.v1.images` — image service
- `io.containerd.cri.v1.runtime` — runtime service (sandboxes, containers)

…and the **gRPC service that exposes them to `crictl` / kubelet** is:

- `io.containerd.grpc.v1.cri`

All three must be listed as **`ok`** in `ctr plugins ls` for `crictl` (and later, kubelet) to work.

A **restart of containerd is required** after changing `/etc/containerd/config.toml` — plugins are only loaded at process start, not on config reload.

### Commands

```bash
# 1. Ensure containerd has actually reloaded the current config
sudo systemctl restart containerd
sudo systemctl is-active containerd   # → active

# 2. Check that ALL CRI-related plugins are loaded
sudo ctr plugins ls | grep -E 'cri|grpc'
```

### Expected output (all three CRI lines must be `ok`)

```text
io.containerd.cri.v1     images    -              ok
io.containerd.cri.v1     runtime   linux/amd64    ok
io.containerd.grpc.v1    cri       -              ok
```

If the **third line (`io.containerd.grpc.v1.cri`) is missing**, `crictl` will fail with:

```
cni plugin not initialized
```

…or the socket will not respond to `RuntimeService` calls at all. See the [Troubleshooting](#troubleshooting-crictl-info-fails-or-grpcv1cri-missing) section below.

### Confirm with `crictl info`

```bash
sudo crictl info
```

A healthy output looks like (abbreviated):

```json
{
  "status": {
    "conditions": [
      { "type": "RuntimeReady", "status": true },
      { "type": "NetworkReady", "status": false,
        "reason": "NetworkPluginNotReady",
        "message": "Network plugin returns error: cni plugin not initialized" }
    ]
  }
}
```

- **`RuntimeReady: true`** — containerd's CRI gRPC service is up. ✅
- **`NetworkReady: false`** — **expected** at this stage; no CNI is installed yet. This will flip to `true` after Cilium/Calico/Flannel is deployed post-`kubeadm init`.

Also confirm the config file contains the gRPC CRI section:

```bash
sudo grep -nE '^\[plugins|grpc\.v1\.cri|cri\.v1' /etc/containerd/config.toml
```

Expected key line:

```toml
[plugins.'io.containerd.grpc.v1.cri']
```

---

## Step 4 — Install kubeadm & kubelet (nodes only)

> **Applies to:** cp01, cp02, cp03, worker1, worker2
> **Not applicable to:** bastion

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

## Step 5 — Install kubectl (bastion only)

> **Applies to:** bastion
> **Not applicable to:** any Kubernetes node

The bastion is **not** a Kubernetes node. It only needs `kubectl` to talk to the cluster API. We standardize on the **same 1.35.8** used across the cluster.

### Script: `install-kubectl.sh`

```bash
#!/bin/bash
set -e

K8S_VERSION="1.35"
K8S_PACKAGE_VERSION="1.35.8-1.1"

echo "=========================================="
echo " Kubernetes kubectl ${K8S_PACKAGE_VERSION}"
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
# 5. Install EXACT kubectl version
# --------------------------------------------------

sudo apt-get install -y kubectl=${K8S_PACKAGE_VERSION}

# --------------------------------------------------
# 6. Hold kubectl
# --------------------------------------------------

sudo apt-mark hold kubectl

# --------------------------------------------------
# 7. Verification
# --------------------------------------------------

echo
echo "========== kubectl version =========="

kubectl version --client

echo
echo "========== Held packages =========="

apt-mark showhold

echo
echo "=========================================="
echo " kubectl installation completed."
echo "=========================================="
```

### Step-by-Step Breakdown

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Remove old `kubernetes.list` and keyring | Clean slate, avoid conflicting repos |
| 2 | Install `ca-certificates`, `curl`, `gpg` | Prerequisites for fetching the signing key |
| 3 | Fetch & dearmor the Kubernetes v1.35 `Release.key` | Verifies authenticity of K8s packages |
| 4 | Add the v1.35 `pkgs.k8s.io` repo | Source for `kubectl` |
| 5 | Install **exact** `kubectl=1.35.8-1.1` | Match cluster version |
| 6 | `apt-mark hold kubectl` | Prevent unintended upgrades |
| 7 | Print client version + held packages | Confirm the install |

### Expected Result

```
Client Version: v1.35.8
```

And:

```bash
apt-mark showhold
```

should show:

```
kubectl
```

---

## Verification

After running the appropriate scripts, you should see output similar to the following.

### On a Kubernetes node (cp/worker)

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

--- crictl version ---
crictl version v1.35.0

--- CRI plugins ---
io.containerd.cri.v1     images    -              ok
io.containerd.cri.v1     runtime   linux/amd64    ok
io.containerd.grpc.v1    cri       -              ok

--- kubeadm version ---
kubeadm version: &version.Info{Major:"1", Minor:"35", GitVersion:"v1.35.8", ...}

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

### On the bastion

```text
========== kubectl version ==========
Client Version: v1.35.8
Kustomize Version: v5.x.x

========== Held packages ==========
kubectl
```

### Manual Verification Commands

```bash
# containerd (nodes only)
containerd --version
dpkg -l | grep containerd.io
grep SystemdCgroup /etc/containerd/config.toml
grep -i pause     /etc/containerd/config.toml
systemctl is-active containerd

# CRI + crictl (nodes only)
ctr plugins ls | grep -E 'cri|grpc'
crictl --version
sudo crictl info
sudo crictl images

# Kubernetes tools (nodes only)
kubeadm version
kubelet --version
apt-mark showhold
systemctl is-enabled kubelet

# kubectl (bastion only)
kubectl version --client
apt-mark showhold
```

### Full Version Snapshot

A quick one-liner to capture every relevant version on any host:

```bash
{
  echo "=== OS ===";           lsb_release -d
  echo "=== containerd ===";   containerd --version 2>/dev/null || echo "not installed"
  echo "=== crictl ===";       crictl --version 2>/dev/null || echo "not installed"
  echo "=== kubeadm ===";      kubeadm version -o short 2>/dev/null || echo "not installed"
  echo "=== kubelet ===";      kubelet --version 2>/dev/null || echo "not installed"
  echo "=== kubectl ===";      kubectl version --client -o yaml 2>/dev/null | grep gitVersion || echo "not installed"
  echo "=== CRI plugins ===";  ctr plugins ls 2>/dev/null | grep -E 'cri|grpc' || true
  echo "=== held ===";         apt-mark showhold
} | tee versions-$(hostname).txt
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
- **kubeconfig on the bastion** — after `kubeadm init`, copy `/etc/kubernetes/admin.conf` to `~/.kube/config` on the bastion.
- **HAProxy / DNS / controlPlane endpoint** — must be provisioned and configured separately.

---

## Next Steps

### On Kubernetes nodes

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

4. **Initialize the first control plane** (cp01):
   ```bash
   sudo kubeadm init --control-plane-endpoint "<HAProxy/DNS>:6443" --pod-network-cidr=10.244.0.0/16
   ```

5. **Join the remaining control planes** (cp02, cp03) with the join command that includes `--control-plane`.

6. **Join the workers** (worker1, worker2) with the standard join command.

7. **Install a CNI plugin** — e.g. Flannel:
   ```bash
   kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
   ```

### On the bastion

1. **Copy the kubeconfig** from the first control plane:
   ```bash
   mkdir -p ~/.kube
   scp <user>@cp01:/etc/kubernetes/admin.conf ~/.kube/config
   chmod 600 ~/.kube/config
   ```

2. **Verify connectivity**:
   ```bash
   kubectl cluster-info
   kubectl get nodes
   kubectl get pods -A
   ```

---

## Troubleshooting: `crictl info` fails or `grpc.v1.cri` missing

This is the single most common post-install issue with containerd 2.x. Symptom:

```text
$ sudo crictl info
FATA[0000] validate service connection: CRI v1 runtime API is not implemented for endpoint "unix:///run/containerd/containerd.sock":
rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService
```

…or the plugin is simply absent from `ctr plugins ls`:

```text
$ sudo ctr plugins ls | grep -E 'cri|grpc'
io.containerd.cri.v1     images    -              ok
io.containerd.cri.v1     runtime   linux/amd64    ok
# <-- io.containerd.grpc.v1  cri  ...  IS MISSING
```

### Root cause

In containerd **2.x**, `/etc/containerd/config.toml` **must** contain:

```toml
[plugins.'io.containerd.grpc.v1.cri']
  stream_server_address = '127.0.0.1'
  ...
```

If the config file is correct **but the running process was started before the file was fixed**, the plugin will not load. containerd reads its plugins **once at process start** — it does not hot-reload `config.toml`.

### Fix (safe during node prep)

```bash
# 1. Reload the currently-installed config into the running process
sudo systemctl restart containerd

# 2. Verify the service is up
sudo systemctl is-active containerd
# → active

# 3. Confirm all three CRI plugins are present
sudo ctr plugins ls | grep -E 'cri|grpc'
```

Expected — **three** lines, all `ok`:

```text
io.containerd.cri.v1     images    -              ok
io.containerd.cri.v1     runtime   linux/amd64    ok
io.containerd.grpc.v1    cri       -              ok
```

### Confirm with crictl

```bash
sudo crictl info
```

Healthy:

```json
"status": {
  "conditions": [
    { "type": "RuntimeReady",             "status": true  },
    { "type": "NetworkReady",             "status": false,
      "reason": "NetworkPluginNotReady",
      "message": "Network plugin returns error: cni plugin not initialized" }
  ]
}
```

- `RuntimeReady: true` → ✅ the CRI gRPC service is running.
- `NetworkReady: false` → **expected** until a CNI plugin is installed after `kubeadm init`.

### Why this is safe now

We are still in the **node preparation phase** — no Kubernetes workloads are running yet. Restarting containerd at this point has no impact on a live cluster. This is exactly why `crictl` is tested **before** `kubeadm init`: so misconfigurations are caught while the blast radius is zero.

> **Production warning:** once a node is part of a running cluster, restarting containerd will briefly disrupt running pods on that node. Drain the node first (`kubectl drain <node> --ignore-daemonsets`) or tolerate the momentary disruption.

### If the plugin is still missing after a restart

Check the config file itself:

```bash
sudo grep -nE '^\[plugins|grpc\.v1\.cri' /etc/containerd/config.toml
```

If `[plugins.'io.containerd.grpc.v1.cri']` is absent, the config was generated by an older containerd (1.x). Regenerate it with the installed binary:

```bash
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Re-apply our two required edits:
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.10"#' /etc/containerd/config.toml

sudo systemctl restart containerd
```

Also check that the CRI plugin is not disabled:

```bash
grep disabled_plugins /etc/containerd/config.toml
# Expected: disabled_plugins = []
```

If `cri` appears in `disabled_plugins`, remove it and restart.

---

## Notes & Caveats

- **Architecture is hardcoded to `amd64`.** For arm64, change the containerd repo line to `arch=arm64` and download `crictl-${VERSION}-linux-arm64.tar.gz`.
- **Ubuntu 24.04 / Noble only.** Different Ubuntu releases require a different repo suite (`jammy`, `focal`, etc.).
- **`apt-mark hold` is intentional.** To upgrade later, `apt-mark unhold`, install the new pin, and re-hold.
- **`crictl` is not held by apt.** It is a static binary in `/usr/local/bin`. Version upgrades are manual — download the new tarball.
- **`crictl` minor version should match the Kubernetes minor** (here, `v1.35.x` ↔ K8s `v1.35.x`).
- **Containerd restarts are required after config changes.** The daemon does not hot-reload `config.toml`; only `systemctl restart containerd` (or `SIGHUP` with some builds) picks up changes.
- **Idempotency.** The Kubernetes scripts wipe and re-add their repo each run. The containerd script is not fully idempotent (`apt-mark hold` will simply report "already held"). The crictl script will overwrite `/usr/local/bin/crictl` and `/etc/crictl.yaml`.
- **`set -e`** is enabled in all scripts — any failing command aborts the script. Read the output carefully if something stops early.
- **Matching versions matters.** `kubeadm`, `kubelet`, and `kubectl` should all be the **same minor version** (here `1.35.x`), and never more than one minor ahead/behind the control plane.
- **Do not start `kubelet` manually** before `kubeadm init` or `kubeadm join`. It is enabled for boot, but `kubeadm` orchestrates its first real start.
- **Bastion is not a node.** Do not install `containerd`, `crictl`, `kubelet`, or `kubeadm` there. `kubectl` is sufficient and preferred for a hardened admin host.
- **Repositories change over time.** Always confirm with `apt-cache madison` / `apt-cache policy` / GitHub API that your pinned version still exists before running the scripts in a new environment.

---

## File Layout

```
.
├── README.md                  # this document
├── install-containerd.sh      # Step 1 — K8s nodes
├── install-crictl.sh          # Step 2 — K8s nodes
├── install-kube-tools.sh      # Step 4 — K8s nodes
└── install-kubectl.sh         # Step 5 — bastion
```

Run the appropriate scripts on each host:

### On Kubernetes nodes (in order)

```bash
chmod +x install-containerd.sh install-crictl.sh install-kube-tools.sh
./install-containerd.sh
./install-crictl.sh

# Verify CRI before proceeding (Step 3)
sudo systemctl restart containerd
sudo ctr plugins ls | grep -E 'cri|grpc'
sudo crictl info

./install-kube-tools.sh
```

### On the bastion

```bash
chmod +x install-kubectl.sh
./install-kubectl.sh
```

---
