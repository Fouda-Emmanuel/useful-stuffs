# Kubernetes Kubeadm Cluster on AWS — Bastion, 1 CP, 3 Workers, Containerd, Calico, VXLAN

**Versions used in this build:** Kubernetes v1.32.13 · containerd 2.2.1 · Calico v3.30.2 · Ubuntu 24.04.4 LTS
**Topology:** 1 control plane, 3 workers, 1 bastion, 3 AZs
**Architecture:** `kubectl` only on the bastion — no cluster node ever has it

> **If you only read one section before you start:** read [14.3](#143--cross-subnet-pod-networking-broken-the-big-one) first.
> It's the bug most likely to bite you, and it makes the cluster look perfectly healthy while pod traffic silently fails.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Network / Security Group Design](#3-network--security-group-design)
4. [Step 1 — Base OS Prep (all nodes)](#4-step-1--base-os-prep-all-nodes)
5. [Step 2 — Install containerd](#5-step-2--install-containerd)
6. [Step 3 — Install kubelet + kubeadm](#6-step-3--install-kubelet--kubeadm)
7. [Step 4 — Install kubectl (bastion only)](#7-step-4--install-kubectl-bastion-only)
8. [Step 5 — Initialize the Control Plane](#8-step-5--initialize-the-control-plane)
9. [Step 6 — Configure kubectl on the Bastion](#9-step-6--configure-kubectl-on-the-bastion)
10. [Step 7 — Install Calico CNI](#10-step-7--install-calico-cni)
11. [Step 8 — Join the Workers](#11-step-8--join-the-workers)
12. [Step 9 — Verify the Cluster](#12-step-9--verify-the-cluster)
13. [Troubleshooting](#13-troubleshooting)
14. [Production Gaps](#14-production-gaps)
15. [Reference: Ports and Protocols](#15-reference-ports-and-protocols)
16. [What I'd Do Differently Next Time](#16-what-id-do-differently-next-time)

---

## 1. Architecture Overview

```
                     ┌──────────────────────┐
                     │       Bastion         │
                     │  kubectl + kubeconfig │
                     │  (admin entry point)  │
                     └──────────┬───────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        HTTPS 6443              │           SSH 22
        (kubectl)               │           (admin)
                │               │               │
                ▼               │               ▼
        ┌─────────────────────────────────────────────┐
        │        CONTROL PLANE (10.90.11.18)           │
        │        AZ-1 / Subnet 10.90.11.0/24           │
        │                                               │
        │  Static pods:                                 │
        │    etcd, kube-apiserver,                      │
        │    kube-controller-manager, kube-scheduler    │
        │                                               │
        │  DaemonSets (every node):                     │
        │    calico-node, kube-proxy, csi-node-driver   │
        └─────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        ┌──────────┐      ┌──────────┐      ┌──────────┐
        │ worker1  │      │ worker2  │      │ worker3  │
        │ 10.90.11 │      │ 10.90.12 │      │ 10.90.13 │
        │  AZ-1    │      │  AZ-2    │      │  AZ-3    │
        │ kubelet  │      │ kubelet  │      │ kubelet  │
        │containerd│      │containerd│      │containerd│
        └──────────┘      └──────────┘      └──────────┘
```

**Key design decisions and their rationale:**

| Decision | Why |
|---|---|
| kubectl only on bastion | Reduces attack surface. Cluster-admin creds on one hardened box, not scattered across nodes. |
| Control plane separate from workers | Standard separation. Allows independent scaling, upgrades, and taints. |
| Nodes in different subnets/AZs | Fault isolation. AZ failure doesn't take the whole cluster. |
| Single control plane (for now) | Acceptable for a lab/learning build. Production should have ≥ 3 control planes across AZs. |
| VXLANCrossSubnet CNI mode | Same-subnet pod traffic avoids encapsulation overhead; cross-subnet gets VXLAN. |
| BGP disabled | No routed fabric; VXLAN-only overlay is simpler and works in any VPC. |
| Same OS + runtime across nodes | Prevents version drift that silently breaks things. |
| Pinned package versions | Avoids surprise upgrades breaking the cluster. |

---

## 2. Prerequisites

### Infrastructure

- AWS VPC with subnets across at least 3 AZs
- 5 EC2 instances:
  - Bastion (any small size; t3.micro is fine)
  - Control plane (≥ 2 vCPU, ≥ 4 GB RAM)
  - 3 workers (≥ 2 vCPU, ≥ 4 GB RAM each)
- **Same Ubuntu AMI for all 4 Kubernetes nodes** (the bastion can differ)
- One SSH key pair, with the bastion able to reach every node

### OS

**Ubuntu 24.04 LTS** (codename `noble`) on all 4 Kubernetes nodes. Do not mix
Ubuntu versions across nodes — this silently breaks containerd/kubelet
version matching (see [14.6](#146-containerd-version-drift-across-nodes)).

**Verify on every node:**
```bash
cat /etc/os-release | grep -E 'PRETTY_NAME|VERSION_ID|VERSION_CODENAME'
```
Expected on all 4 nodes:
```
PRETTY_NAME="Ubuntu 24.04.4 LTS"
VERSION_ID="24.04"
VERSION_CODENAME=noble
```

### Sizing

| Node | Minimum | Recommended |
|---|---|---|
| Control plane | 2 vCPU / 2 GB | 2 vCPU / 4 GB |
| Worker | 2 vCPU / 2 GB | 2 vCPU / 4 GB |
| Bastion | 1 vCPU / 1 GB | t3.micro |

> ⚠️ `kubeadm`'s preflight checks fail on a 1-vCPU control plane. Resize the
> instance rather than bypassing the check with `--ignore-preflight-errors=NumCPU`.

### SSH layout

- Laptop → bastion (bastion's public/reachable IP)
- Bastion → all 4 Kubernetes nodes (private IPs)
- Every node's SG allows SSH **only** from the bastion's SG — never from your laptop directly

**OPTIONAL — SSH straight to a node through the bastion, without a manual second hop:**
```
# ~/.ssh/config on your laptop
Host bastion
    HostName <BASTION_PUBLIC_IP>
    User ubuntu
    IdentityFile ~/.ssh/kubeadm-demo

Host controlplane worker1 worker2 worker3
    User ubuntu
    IdentityFile ~/.ssh/kubeadm-demo
    ProxyJump bastion
```
Fill in each host's real private IP under its own `Host` block first (add a
`HostName` line per host). With this in place, `ssh worker1` from your laptop
transparently jumps through the bastion. Skip this if you're fine SSHing into
the bastion first and then SSHing again from there.

---

## 3. Network / Security Group Design

Create three security groups in the same VPC: `bastion-sg`,
`controlplane-sg`, `worker-sg`. Allow all egress on all three.

**`bastion-sg` (inbound):**

| Protocol | Port | Source | Purpose |
|---|---|---|---|
| TCP | 22 | your-admin-IP/32 | SSH from your laptop |

**`controlplane-sg` (inbound):**

| Protocol | Port | Source | Purpose |
|---|---|---|---|
| TCP | 22 | `bastion-sg` | SSH from bastion |
| TCP | 6443 | `bastion-sg` | kubectl from bastion |
| TCP | 6443 | `worker-sg` | kubelet → API server (workers) |
| TCP | 2379–2380 | `controlplane-sg` | etcd (only needed for multi-CP; harmless with one) |
| TCP | 10250 | `worker-sg` | API server → kubelet on workers |
| TCP | 10257 | `controlplane-sg` | controller-manager |
| TCP | 10259 | `controlplane-sg` | scheduler |
| TCP | 5473 | `worker-sg` | Typha (Calico) |
| TCP | 5473 | `controlplane-sg` | Typha (self) |
| **UDP** | **4789** | `worker-sg` | **VXLAN (cross-subnet)** |
| **UDP** | **4789** | `controlplane-sg` | **VXLAN (self / same SG)** |

**`worker-sg` (inbound):**

| Protocol | Port | Source | Purpose |
|---|---|---|---|
| TCP | 22 | `bastion-sg` | SSH from bastion |
| TCP | 10250 | `controlplane-sg` | kubelet API from API server |
| TCP | 10250 | `worker-sg` | kubelet API from workers |
| TCP | 5473 | `controlplane-sg` | Typha |
| TCP | 5473 | `worker-sg` | Typha |
| TCP | 30000–32767 | `bastion-sg` | NodePort (lab testing only) |
| **UDP** | **4789** | `controlplane-sg` | **VXLAN cross-subnet** |
| **UDP** | **4789** | `worker-sg` | **VXLAN worker-to-worker** |

> 🔴 **CRITICAL:** Port 4789 is **UDP**, not TCP. Calico VXLAN uses UDP. This
> is the single most common SG mistake, and it produces a cluster that
> *looks* healthy (nodes Ready, pods Running) while cross-subnet pod
> networking is silently broken. See [14.3](#143--cross-subnet-pod-networking-broken-the-big-one).

### Why these specific rules

- **6443** — the Kubernetes API server. Everything talks to it: kubectl (bastion), kubelets (workers), Calico.
- **10250** — kubelet API. Used by the API server for `kubectl exec`, `kubectl logs`, port-forward, and by metrics-server.
- **2379–2380** — etcd client + peer. Only strictly required for multi-CP, included here for future-proofing.
- **5473** — Calico Typha, a datastore proxy so `calico-node` instances don't all hammer the API server directly.
- **4789/UDP** — the VXLAN tunnel. Required for cross-subnet pod-to-pod traffic in `VXLANCrossSubnet` mode.
- **30000–32767** — the default NodePort range. Not exposed to the internet in production — only to the bastion SG, for testing.

---

## 4. Step 1 — Base OS Prep (all nodes)

Run on **all 4 Kubernetes nodes**: control plane + worker1 + worker2 +
worker3. **Not** the bastion.

```bash
# Update the system
sudo apt-get update
sudo apt-get upgrade -y

# Disable swap (required by kubelet)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules for container networking
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

# Sysctls required for pod/Service networking
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

**Why:**
- Swap off → kubelet's resource management assumes no swap; it refuses to start otherwise, and swap makes QoS/OOM handling non-deterministic.
- `overlay` → enables the overlay filesystem containerd's default snapshotter uses for container images.
- `br_netfilter` + bridge sysctls → let iptables see bridged (pod-to-pod) traffic. Without this, Service routing silently fails.
- `ip_forward=1` → lets the node route packets between interfaces, which pod networking depends on.

**Verify:**
```bash
swapon --show                           # empty
lsmod | grep -E 'overlay|br_netfilter'  # both listed
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward   # both = 1
nproc                                   # >= 2
```

### OPTIONAL — clearer hostnames

Skip this if your instances already have names you're happy with — it only
affects the readability of `kubectl get nodes`, nothing functional.

```bash
# On the control plane
sudo hostnamectl set-hostname controlplane
exec bash
```
```bash
# On each worker (adjust the name per worker)
sudo hostnamectl set-hostname worker1
exec bash
```

### OPTIONAL — /etc/hosts entries

Lets you `ssh worker1` (etc.) by name from any node. Doesn't affect
Kubernetes at all.

```bash
sudo tee -a /etc/hosts <<EOF
10.90.11.18   controlplane
10.90.11.250  worker1
10.90.12.148  worker2
10.90.13.91   worker3
EOF
```

---

## 5. Step 2 — Install containerd

Run on **all 4 Kubernetes nodes**. **Not** the bastion.

```bash
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Align containerd with kubelet's expectations
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl status containerd --no-pager
```

**Why:**
- containerd is the container runtime kubelet talks to.
- `SystemdCgroup = true` matches kubelet's cgroup driver (systemd, the default since Kubernetes 1.24+). If these don't match, pods start but are unstable — random restarts, wrong resource accounting, eventually a broken node. One of the most common silent cluster killers, so it's worth verifying explicitly rather than trusting the default.
- Pinning the pause image (`pause:3.9`) keeps every node using the same sandbox image, avoiding needless pod sandbox churn.

**Verify:**
```bash
containerd --version
sudo grep SystemdCgroup /etc/containerd/config.toml   # expect: = true
systemctl is-active containerd                        # active
systemctl is-enabled containerd                       # enabled
```

### OPTIONAL — pin the containerd version

Only needed if you notice version drift between nodes (see
[14.6](#146-containerd-version-drift-across-nodes)).

```bash
sudo apt-mark hold containerd
```

---

## 6. Step 3 — Install kubelet + kubeadm

Run on **all 4 Kubernetes nodes**. **Not** the bastion — no `kubectl` here.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm
sudo apt-mark hold kubelet kubeadm
sudo systemctl enable kubelet
```

**Why:**
- The Kubernetes project maintains its own apt repo at `pkgs.k8s.io` — use it instead of Ubuntu's, so you get exactly the upstream minor version you want and can pin it deliberately.
- `kubeadm` bootstraps the cluster; `kubelet` runs pods on the node.
- **No `kubectl` here on purpose** — it belongs on the bastion only (Step 4).
- Holding the versions (`apt-mark hold`) stops a routine `apt upgrade` from silently bumping `kubelet` without `kubeadm` (or vice versa) and breaking the cluster.
- `kubelet` is enabled but not started yet — it has no config until `kubeadm init`/`join` runs, and would just crash-loop if started early.

**Verify:**
```bash
kubeadm version
kubelet --version
apt-mark showhold                # kubelet, kubeadm
systemctl is-enabled kubelet     # enabled
which kubectl                   # prints nothing — confirms it's not installed here
```

---

## 7. Step 4 — Install kubectl (bastion only)

Run on the **bastion**. **Never on the cluster nodes.**

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
sudo apt-mark hold kubectl
```

**Why:**
- `kubectl` is the admin CLI — it just talks to the API server over the network, so it only needs to live on one hardened, SSH-only host.
- Holding the version keeps it in lockstep with the cluster's Kubernetes version.

**Verify:**
```bash
kubectl version --client       # v1.32.13
which kubelet                  # prints nothing
which kubeadm                  # prints nothing
```
`kubectl version` will still complain about `localhost:8080` being refused —
expected, there's no kubeconfig yet.

### OPTIONAL — bash completion + a short alias

Purely a shell convenience. Skip it if you don't want `~/.bashrc` touched.

```bash
sudo apt-get install -y bash-completion
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

---

## 8. Step 5 — Initialize the Control Plane

Run on the **control plane only**.

```bash
# Pre-flight sanity check
swapon --show
systemctl is-active containerd
grep SystemdCgroup /etc/containerd/config.toml

# Replace with your control plane node's private IP
sudo kubeadm init \
  --control-plane-endpoint=<CP_PRIVATE_IP>:6443 \
  --apiserver-advertise-address=<CP_PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16
```

**Why:**
- `--control-plane-endpoint` — the address other nodes and clients use to reach the API server. It's baked into every kubeconfig and kubelet config generated from here on.
- `--apiserver-advertise-address` — the local IP the API server actually binds to.
- `--pod-network-cidr` — the CIDR Calico will allocate pod IPs from; must match what you set in the CNI installation (Step 7), and must not overlap your VPC CIDR, any node subnet CIDR, or the default service CIDR (`10.96.0.0/12`).
- Using the control plane's private IP directly is fine for a single-CP build. For a future HA upgrade, `--control-plane-endpoint` should ideally be a load-balancer DNS name or a stable VIP instead, so client configs never need to change if the CP is replaced.

**Save the join command** printed at the end of the output — you'll need it
in Step 8:
```
kubeadm join <CP_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```
The token expires after 24 hours; if you lose it or it expires, regenerate
with `sudo kubeadm token create --print-join-command`.

> ⚠️ **Skip** the `mkdir -p $HOME/.kube && sudo cp admin.conf ...` lines that
> `kubeadm init`'s own output suggests running next. That copies
> cluster-admin credentials onto the control plane's user account —
> unnecessary here, since `kubectl` only ever runs from the bastion (Step 6).

**Verify:**
```bash
sudo systemctl status kubelet --no-pager
sudo crictl pods 2>/dev/null | grep -E 'etcd|apiserver|scheduler|controller'
```
You should see the four static pods (etcd, apiserver, controller-manager,
scheduler) running.

### OPTIONAL — bypass the 2-vCPU check (tiny/demo VMs only)

Only use this if your control plane genuinely can't be resized. It skips a
real check, not a cosmetic one — startup will be slower and less stable.

```bash
sudo kubeadm init \
  --control-plane-endpoint=<CP_PRIVATE_IP>:6443 \
  --apiserver-advertise-address=<CP_PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16 \
  --ignore-preflight-errors=NumCPU
```

---

## 9. Step 6 — Configure kubectl on the Bastion

**On the control plane:**
```bash
sudo cp /etc/kubernetes/admin.conf /tmp/admin.conf
sudo chown $USER:$USER /tmp/admin.conf
chmod 600 /tmp/admin.conf
```

**On the bastion:**
```bash
mkdir -p ~/.kube
scp ubuntu@<CP_PRIVATE_IP>:/tmp/admin.conf ~/.kube/config
chmod 600 ~/.kube/config
grep server ~/.kube/config      # should show https://<CP_PRIVATE_IP>:6443
kubectl get nodes                # control plane shows NotReady — expected, no CNI yet
```

**Why:** `/etc/kubernetes/admin.conf` is the cluster-admin kubeconfig. It's
staged briefly on the control plane, pulled onto the bastion, and then
removed from the control plane — so no long-lived copy of it exists anywhere
but the bastion.

**Back on the control plane, clean up the temp copy:**
```bash
rm /tmp/admin.conf
```

If `scp` fails with `Permission denied (publickey)`, see
[13.8](#138-scp-from-bastion-to-control-plane-fails-with-permission-denied-publickey).

---

## 10. Step 7 — Install Calico CNI

Run from the **bastion**.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - cidr: 192.168.0.0/16
      natOutgoing: Enabled
      blockSize: 26
      encapsulation: VXLANCrossSubnet
      nodeSelector: all()
EOF
```

**Why (field by field):**

| Field | Value | Why |
|---|---|---|
| `bgp` | `Disabled` | We're using a VXLAN overlay, not routed fabric. Leaving BGP on causes BIRD readiness failures. |
| `cidr` | `192.168.0.0/16` | Must match `--pod-network-cidr` from `kubeadm init`. |
| `natOutgoing` | `Enabled` | SNAT so pods reaching the internet use the node's IP. |
| `blockSize` | `26` | 64 pod IPs per node; increase if you need more pods per node. |
| `encapsulation` | `VXLANCrossSubnet` | Same-subnet: no encapsulation. Cross-subnet: VXLAN. |

**Verify:**
```bash
kubectl get pods -n calico-system -w
# wait until every pod is Running and calico-node is 1/1
kubectl get nodes
# control plane should flip to Ready
kubectl get installation default -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'
# Disabled
```

---

## 11. Step 8 — Join the Workers

**On the control plane**, get a fresh join command (tokens expire after 24h):
```bash
sudo kubeadm token create --print-join-command
```

**On each worker**, confirm it's ready, then join:
```bash
# Pre-flight sanity check
nproc
swapon --show
systemctl is-active containerd
grep SystemdCgroup /etc/containerd/config.toml
kubeadm version
kubelet --version

# Join (replace with the values printed on the control plane)
sudo kubeadm join <CP_PRIVATE_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

**Why:** the pre-flight check catches a mismatched or unprepared worker
before you spend time debugging a failed join — expect ≥2 vCPU, no swap,
containerd active, `SystemdCgroup = true`, and kubeadm/kubelet versions that
match the control plane exactly.

**Verify (from the bastion):**
```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep calico-node
```
All 4 nodes `Ready`; 4 `calico-node` pods at `1/1`.

---

## 12. Step 9 — Verify the Cluster

Run from the **bastion**.

### Cluster health

```bash
kubectl get nodes -o wide
kubectl get ds -A
kubectl get pods -n kube-system
kubectl get --raw='/readyz?verbose'
```
Expect all nodes `Ready` with identical `VERSION`/`OS-IMAGE`/`CONTAINER-RUNTIME`,
`DESIRED = CURRENT = READY` on every DaemonSet, and all `kube-system` pods
`Running`.

### Cross-node pod-to-pod networking (the real test)

```bash
kubectl run nettest --image=nicolaka/netshoot --rm -it --restart=Never -- bash
```
Inside, curl the IPs of pods running on *other* nodes (get their IPs from
`kubectl get pods -o wide` in another terminal):
```bash
for ip in <IP1> <IP2> <IP3>; do
  echo -n "$ip -> "; curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 http://$ip
done
```
Expect all `200`. This is the step that actually exercises VXLAN — see
[13.3](#133--cross-subnet-pod-networking-broken-the-big-one) if it fails.

### DNS resolution

```bash
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
```
Expect an answer from the cluster DNS service IP (usually `10.96.0.10`).

### NodePort from all workers

```bash
kubectl create deploy web --image=nginx --replicas=6
kubectl expose deploy/web --port=80 --type=NodePort

NP=$(kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}')
for ip in <worker1-ip> <worker2-ip> <worker3-ip>; do
  echo -n "$ip -> "; curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 -I http://$ip:$NP
done
```
Expect all `200`. Requires `worker-sg` to allow TCP `30000–32767` from
`bastion-sg`.

### ✅ Final Checkpoint

```bash
kubectl get nodes -o wide                                    # 4 Ready, matched
kubectl get pods -A | grep -v Running | grep -v Completed    # nothing output
kubectl get ds -A                                            # full coverage
```
Cluster is done.

### OPTIONAL — confirm the cluster survives a reboot

Not required to consider the build finished, but the clearest proof it's
solid. Do this on the control plane, then each worker, one at a time:

```bash
sudo reboot
```
After each reboot, wait 2–3 minutes, then from the bastion:
```bash
kubectl get nodes
kubectl get pods -A
```
Everything should return healthy with **one restart each**, timestamped at
the reboot — see [13.13](#1313-pods-show-restarts-1-after-stoppingstarting-instances)
for what that looks like versus a genuine crash loop.

---

## 13. Troubleshooting

Real problems hit while building this cluster, with diagnosis and fix for each.

### 13.1 Pods stuck `Pending` with "0/1 nodes available: untolerated taint"

**Symptom:**
```
Warning  FailedScheduling  0/1 nodes are available: 1 node(s) had untolerated taint
{node-role.kubernetes.io/control-plane: }.
```
**Cause:** the only node is the control plane, and it's tainted against
ordinary workloads.

**Fix:** join worker nodes, or (single-node clusters only):
```bash
kubectl taint nodes controlplane node-role.kubernetes.io/control-plane:NoSchedule-
```
**Why it's not a bug:** the taint exists so user workloads don't compete with
the API server, etcd, and other control-plane processes for resources.

---

### 13.2 CoreDNS stuck `Pending`, node `NotReady`

**Symptom:** immediately after `kubeadm init`, before the CNI is installed —
CoreDNS is `Pending` and the control plane is `NotReady`.

**Cause:** no CNI yet, so pods can't get pod-network IPs.

**Fix:** install Calico (Step 7). Resolves itself once Calico is running.

---

### 13.3 🔴 Cross-subnet pod networking broken (the big one)

**Symptom:** the cluster looks entirely healthy — all nodes `Ready`, all pods
`Running` — but pod-to-pod traffic between specific node pairs fails.
NodePort services behave inconsistently: same node works, a different node
hangs.

**Example from this build:**
```
pod on worker3 → ✅ 200   (same subnet)
pod on worker1 → ❌ 000
pod on worker2 → ❌ 000
```

**Diagnose:**
```bash
kubectl get ippools -o wide
# VXLANMODE column should say CrossSubnet

kubectl run nettest --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# inside: curl each pod IP from a different node and check the response
```

**Root cause:** in `VXLANCrossSubnet` mode, Calico encapsulates cross-subnet
traffic in VXLAN, which runs over **UDP port 4789**. If any node's security
group blocks UDP 4789 from another node, that node pair can't talk.

**The exact mistake we hit:** the SG rules for 4789 were set as **TCP**
instead of **UDP**. TCP 4789 does nothing for VXLAN — an "open but useless"
rule — so all VXLAN traffic was silently dropped.

**Fix:** on every node's security group, confirm these inbound rules exist —
and delete any TCP 4789 rules added by mistake:

| Protocol | Port | Source |
|---|---|---|
| **UDP** | **4789** | every other node's SG |
| **UDP** | **4789** | the same SG (for intra-SG nodes) |

**Why it's easy to miss:** the control plane, etcd, kubelet, and CoreDNS all
still work (host-network or same-subnet traffic). Only cross-subnet
pod-to-pod traffic breaks. If you never test that specific path, you won't
notice — which is exactly why it's part of Step 9's verification.

---

### 13.4 `calico-node` shows `Running (0/1)` — readiness failing

**Symptom:** logs mention "Error querying BIRD" or "BGP not established,"
even though this cluster uses VXLAN and doesn't need BGP.

**Cause:** the Tigera operator defaults to BGP enabled. In a VXLAN-only
cluster, BIRD tries to establish BGP sessions that never come up, and
`calico-node` fails its readiness check.

**Fix:**
```bash
kubectl get installation.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'
```
If it prints `Enabled` or nothing:
```bash
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'
```
If it doesn't reconcile automatically:
```bash
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node
```

---

### 13.5 `calico-node` not ready — "Failed to connect to Typha"

**Cause:** Typha listens on TCP 5473; the SG doesn't allow it between nodes.

**Fix:** allow TCP 5473 between `worker-sg` and `controlplane-sg` (both
directions).

**Verify:**
```bash
kubectl -n calico-system get svc,endpoints -l k8s-app=calico-typha
kubectl -n calico-system logs ds/calico-node -c calico-node | grep -i typha
```

---

### 13.6 containerd version drift across nodes

**Symptom:** `containerd --version` differs between nodes.

**Cause:** `apt install containerd` installs whatever the node's configured
repos currently offer — nodes on different Ubuntu versions get different
containerd versions.

**Fix:** standardize the OS. Don't try to downgrade containerd on the odd
node out — rebuild it from the same AMI as the others.

---

### 13.7 Kubernetes package version drift

**Symptom:** `kubelet --version` differs between nodes.

**Fix:**
```bash
sudo apt-mark unhold kubelet kubeadm
sudo apt-get install -y kubelet=<version> kubeadm=<version>
sudo apt-mark hold kubelet kubeadm
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
**Prevention:** pin versions with `apt-mark hold` at install time (Step 3).

---

### 13.8 `scp` from bastion to control plane fails with "Permission denied (publickey)"

**Cause:** the bastion doesn't have an SSH key authorized on the target node.

**Fix (option A) — authorize the bastion's key on the CP:**
```bash
# on the bastion
cat ~/.ssh/id_ed25519.pub
```
```bash
# on the control plane
echo '<paste the bastion public key here>' >> ~/.ssh/authorized_keys
```

**Fix (option B) — skip scp entirely:** `cat /tmp/admin.conf` on the control
plane and paste its contents into `~/.kube/config` on the bastion.

---

### 13.9 NodePort hangs for some nodes but works for others

**Symptom:**
```
curl -I http://worker1:30080   → 200 OK
curl -I http://worker2:30080   → hangs
curl -I http://worker3:30080   → 200 OK
```
**Cause:** under the default `externalTrafficPolicy: Cluster`, a node's
kube-proxy can forward traffic to a pod on a *different* node. If cross-node
VXLAN traffic is broken (see [13.3](#133--cross-subnet-pod-networking-broken-the-big-one)),
requests routed to a remote pod hang. Which node "hangs" looks essentially
random — it depends on which backend kube-proxy picks.

**Fix:** fix the underlying VXLAN path first. Optionally, for production
behavior:
```bash
kubectl patch svc web -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```
Note `Local` isn't a substitute for fixing 13.3 — a node with no local
backend will simply drop the packet under `Local`.

---

### 13.10 `kubectl` returns "localhost:8080 was refused"

**Cause:** no kubeconfig yet, so kubectl falls back to `localhost:8080`.

**Fix:** configure `~/.kube/config` (Step 6). This error is expected before
that step and nothing to worry about.

---

### 13.11 `kubeadm --version` fails with "unknown flag"

**Not a bug** — `kubeadm` doesn't support `--version`; use the subcommand:
```bash
kubeadm version
```

---

### 13.12 Join token expired

**Symptom:**
```
could not find a JWS signature in the cluster-info ConfigMap for token ...
```
**Cause:** kubeadm tokens expire after 24 hours by default.

**Fix (on the control plane):**
```bash
sudo kubeadm token create --print-join-command
```

---

### 13.13 Pods show `RESTARTS 1` after stopping/starting instances

**Not a crash loop.** Stopping and starting an EC2 instance restarts kubelet,
which restarts every pod on that node exactly once. A genuine crash loop
looks like a high, still-incrementing restart count with a recent timestamp
(e.g. `RESTARTS 47 (30s ago)`) — a single restart matching the reboot time is
healthy. Pod IPs (`192.168.x.x`) may also change after a reboot, since Calico
reallocates them; node IPs stay fixed.

---

## 14. Production Gaps

The cluster as built is a solid foundation. These are the next layers.

### High availability

| Gap | Fix |
|---|---|
| Single control plane | Run 3 control planes across 3 AZs; use `kubeadm join --control-plane` on the extras. |
| No API load balancer | Add an NLB in front of the CPs; point `--control-plane-endpoint` at a stable DNS name resolving to it. |

### Networking

| Gap | Fix |
|---|---|
| NodePorts exposed to bastion | Add an ingress controller (nginx-ingress or Traefik), exposed via ALB/NLB. Stop using NodePort for real traffic. |
| No NetworkPolicies | Apply default-deny and allow specific paths — Calico enforces these natively. |
| No mesh | (Optional) Istio/Linkerd for mTLS, observability, traffic shaping. |

### Storage

| Gap | Fix |
|---|---|
| No PersistentVolumes | Install the EBS CSI driver; create a default StorageClass. |
| No snapshot strategy | EBS snapshots via Velero or AWS Backup. |

### Security

| Gap | Fix |
|---|---|
| Cluster-admin for everyone | Per-role ServiceAccounts + RBAC; scoped kubeconfigs for bastion users. |
| No admission control | Enable PodSecurity admission; add OPA/Gatekeeper or Kyverno. |
| No audit logging | Enable API server audit logs; ship to CloudWatch/S3. |
| No image scanning | Trivy (or similar) in CI; admission control to block known-vulnerable images. |

### Observability

| Gap | Fix |
|---|---|
| No metrics | Install `metrics-server` for `kubectl top` and HPA. |
| No dashboards | Prometheus + Grafana (kube-prometheus-stack). |
| No log aggregation | Loki + Promtail, or Fluent Bit → CloudWatch/OpenSearch. |
| No tracing | Tempo or Jaeger. |
| No alerting | Alertmanager + PagerDuty/Opsgenie integration. |

### Operations

| Gap | Fix |
|---|---|
| Manual `kubectl apply` | GitOps (Argo CD or Flux). |
| No IaC | Terraform for AWS resources; Ansible for OS config. |
| No backup | Automated etcd snapshots; Velero for cluster resources. |
| No upgrade process | Document a `kubeadm upgrade` runbook; test in staging first. |
| No node auto-repair | Cluster Autoscaler, Karpenter, or ASG health checks. |

**Recommended order:** ingress controller → metrics-server → EBS CSI +
StorageClass → NetworkPolicies → RBAC/ServiceAccounts → observability stack →
etcd/Velero backups → second and third control plane + NLB → GitOps.

---

## 15. Reference: Ports and Protocols

| Component | Port | Protocol | Purpose |
|---|---|---|---|
| kube-apiserver | 6443 | TCP | API server — everything talks to this |
| etcd client | 2379 | TCP | etcd API |
| etcd peer | 2380 | TCP | etcd replication |
| kubelet | 10250 | TCP | kubelet API (exec/logs/port-forward) |
| kube-scheduler | 10259 | TCP | metrics/serving |
| kube-controller-manager | 10257 | TCP | metrics/serving |
| NodePort services | 30000–32767 | TCP | default NodePort range |
| Calico Typha | 5473 | TCP | Calico datastore proxy |
| **Calico VXLAN** | **4789** | **UDP** | **VXLAN encapsulation** |
| Calico BGP | 179 | TCP | BGP (disabled in this setup) |
| CoreDNS | 53 | UDP + TCP | cluster DNS |
| WireGuard (if used) | 51820 | UDP | encrypted overlay |
| metrics-server | 443 | TCP | to the API server |

**Golden rule:** always specify both protocol and port. `4789` alone is
meaningless; `UDP 4789` is what VXLAN actually uses.

---

## 16. What I'd Do Differently Next Time

Honest notes for the next build:

- **Put the control plane alone in its own AZ**, rather than sharing an AZ
  with worker1. As built, an AZ outage takes out the control plane and a
  worker together.
- **Use a DNS name (or a small internal load balancer) for the control-plane
  endpoint from day one**, even with a single CP — it costs nothing now and
  means the CP can be replaced later without touching any client config.
- **Write the SG rules table before creating anything in the console.** The
  TCP/UDP 4789 mistake happened because the rule was added by memory instead
  of checked against a reference table like the one in Section 3.
- **Test cross-subnet pod-to-pod connectivity right after installing
  Calico**, not just at the end — it would have caught the VXLAN issue much
  earlier instead of after workers were joined and an app was deployed.

---
