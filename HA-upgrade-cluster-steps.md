# Kubernetes HA Cluster Upgrade Runbook
## v1.35.8 → v1.36.4 — Complete Step-by-Step Procedure

**Cluster:** faekcorp-lab
**Source document:** Complete Execution Record (2026-10-11 / 2026-10-12)
**Audience:** DevOps engineers, junior SREs, and anyone performing a Kubernetes minor-version upgrade on this cluster (or a similar kube-proxy-free Cilium HA cluster).
**Purpose:** This is the reusable, step-by-step procedure extracted from our actual executed upgrade. Follow this from top to bottom. Every command is annotated with **what it does**, **why we run it**, and **what to watch for**. Traps and gotchas are flagged inline as **Important** notes so you don't repeat our mistakes.

> **READ THIS FIRST — Cluster-Specific Context**
> This cluster is **NOT a standard kubeadm cluster**. It runs **Cilium 1.20.1 in eBPF mode with `KubeProxyReplacement=True`**, and **kube-proxy has been removed** (no DaemonSet, no ConfigMap). Standard kubeadm documentation assumes kube-proxy exists. That single difference caused our main incident during this upgrade. Everything marked **Important** below is a trap specific to this architecture.

> **Administration host:** `kubectl` is **NOT installed on cluster nodes**. Only on the bastion. All `kubectl` commands in this document are run **from the bastion**. Commands run on nodes are marked as such.

---

## Table of Contents

1. [Pre-Upgrade Planning & Version Selection](#1-pre-upgrade-planning--version-selection)
2. [Compatibility Verification](#2-compatibility-verification)
3. [Phase 0 — Pre-Flight Verification](#3-phase-0--pre-flight-verification)
4. [Critical Concept: apply vs node](#4-critical-concept-apply-vs-node)
5. [Phase 1 — Upgrade First Control-Plane (cp01)](#5-phase-1--upgrade-first-control-plane-cp01)
6. [Phase 2 — Upgrade Remaining Control-Planes (cp02, cp03)](#6-phase-2--upgrade-remaining-control-planes-cp02-cp03)
7. [Incident Response: kube-proxy Reinstall](#7-incident-response-kube-proxy-reinstall)
8. [Phase 3 — Upgrade Workers (worker1, worker2)](#8-phase-3--upgrade-workers-worker1-worker2)
9. [Phase 4 — Post-Upgrade Validation](#9-phase-4--post-upgrade-validation)
10. [Bastion kubectl Upgrade](#10-bastion-kubectl-upgrade)
11. [Controlled Reboots](#11-controlled-reboots)
12. [Kernel Update Procedure](#12-kernel-update-procedure)
13. [Final State Checklist](#13-final-state-checklist)
14. [Lessons Learned / What To Do Differently](#14-lessons-learned--what-to-do-differently)
15. [Appendices](#15-appendices)

---

## 1. Pre-Upgrade Planning & Version Selection

### 1.1 Establish Cluster Facts

**Why:** Before touching anything, write down the shape of the cluster. This becomes your reference when something looks wrong mid-upgrade.

| Item | Value (our cluster) |
|------|---------------------|
| Cluster name | faekcorp-lab |
| API HA endpoint | `haproxy.faekcorp.lab:6443` |
| HAProxy IP | 10.70.11.192 |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |
| DNS domain | cluster.local |
| CNI | Cilium 1.20.1 (eBPF, KubeProxyReplacement=True) |
| kube-proxy | **Removed** — no DaemonSet, no ConfigMap |
| etcd | 3.6.6 (stacked, 3 members) |
| Administration host | Bastion (kubectl + helm only) |
| kubectl on nodes? | **No** — bastion only |

### 1.2 Nodes

**Why:** You need to know each node's role and IP so you can reason about quorum, drains, and HAProxy routing.

| Node | Role | Internal IP | OS | containerd | Pre-upgrade K8s |
|------|------|-------------|-----|-----------|-----------------|
| cp01.faekcorp.lab | control-plane | 10.70.21.6 | Ubuntu 24.04.4 | 2.2.6 | v1.35.8 |
| cp02.faekcorp.lab | control-plane | 10.70.31.209 | Ubuntu 24.04.4 | 2.2.6 | v1.35.8 |
| cp03.faekcorp.lab | control-plane | 10.70.41.250 | Ubuntu 24.04.4 | 2.2.6 | v1.35.8 |
| worker1.faekcorp.lab | worker | 10.70.61.85 | Ubuntu 24.04.4 | 2.2.6 | v1.35.8 |
| worker2.faekcorp.lab | worker | 10.70.11.67 | Ubuntu 24.04.4 | 2.2.6 | v1.35.8 |

### 1.3 Choose Target Version

**Why:** Kubernetes upgrades must be **sequential by minor version** — you cannot skip minors (1.35 → 1.37 is forbidden). You also want to pick a *soaked* patch, not the bleeding-edge one.

**Policy applied:** Follow the **Prod N-1 patch policy** — within a minor line, production deployments lag the absolute latest patch by one to absorb soak time.

| Minor | Latest patch | EOL | Notes |
|-------|-------------|-----|-------|
| 1.37 | 1.37.1 | 2027-10-28 | Newest minor — too fresh |
| 1.36 | 1.36.5 | 2027-06-28 | Our target minor |
| 1.35 | 1.35.9 | 2027-02-28 | Our current minor |
| 1.34 | 1.34.12 | 2026-10-27 | Already EOL |

**Chosen version string:** `1.36.4-1.1` (the `-1.1` is the Debian package revision suffix — you must include it when pinning with apt).

**Rationale:**
- 1.36.5 was released ~4 weeks before our upgrade — not enough soak time
- 1.36.4 was released ~2 months before — well-soaked
- Moving *into* a new minor, we want the safest patch in that line, not the newest

**Why not 1.35.9 first?** Patching within the old minor buys nothing when you're about to leave it. `kubeadm upgrade` supports jumping directly from the current patch to the next minor.

### 1.4 Version Skew Rules

**Why:** Kubernetes enforces (and supports) specific skews during a rolling upgrade. Violating them breaks the control plane.

| Component | Allowed skew |
|-----------|--------------|
| kube-apiserver (HA) | All within 1 minor |
| kube-controller-manager / kube-scheduler | ≤ apiserver, prefer same minor |
| kubelet | ≤ apiserver, up to 2 minors behind |
| kubectl | ±1 minor from apiserver |

**Upgrade order that respects these:**
1. All 3 control planes first (apiserver, CM, scheduler move together via static pods)
2. Workers last (kubelet can lag up to 2 minors; we lagged 1)
3. Bastion kubectl at the end

---

## 2. Compatibility Verification

**Why:** Before upgrading Kubernetes, verify that every component underneath it (CRI, CNI, etcd) supports the target version. Upgrading K8s into an unsupported component is the fastest way to brick a cluster.

### 2.1 containerd (CRI)

| K8s version | containerd minimum |
|-------------|--------------------|
| 1.35 | 2.2.0+ |
| **1.36** | **2.3.0+, 2.2.0+** |
| 1.37 | 2.4.0+, 2.3.0+ |

**Our containerd: 2.2.6** → satisfies `2.2.0+`. **No upgrade required.**

### 2.2 Cilium (CNI)

| Cilium version | Supported Kubernetes |
|----------------|----------------------|
| 1.20 | >=1.33 <1.37 |
| 1.19 | >=1.32 <1.36 |
| 1.18 | >=1.30 <1.34 |

**Our Cilium: 1.20.1** → supports 1.33–1.36. **No upgrade required.**

### 2.3 etcd

**Why:** etcd is the cluster's source of truth. Its upgrade is bundled into `kubeadm upgrade` and reported by `kubeadm upgrade plan`.

`kubeadm upgrade plan` reports the target etcd version. Our case: **3.6.6 → 3.6.8** (same minor, patch-level change). Safe.

### 2.4 What We Chose NOT to Upgrade

**Principle: One change at a time.** Every extra moving part multiplies the failure surface.

| Component | Stayed at | Why |
|-----------|-----------|-----|
| containerd | 2.2.6 | Already in supported range; restarting CRI on all 5 nodes adds risk |
| Cilium | 1.20.1 | Already supports 1.36; adding a second moving part multiplies failure surface |

> **Important:** If either of these had been *outside* the supported range, we would have upgraded them **first**, then Kubernetes. See Appendix B and C for those procedures.

---

## 3. Phase 0 — Pre-Flight Verification

**Why:** Capture the cluster state as a baseline for post-upgrade comparison and rollback reference. If something breaks during the upgrade, this is what "normal" looked like.

> **All `kubectl` commands in this section run from the BASTION.**

### 3.1 From the Bastion

```bash
kubectl get nodes -o wide
```
> Lists every node with IP, OS, kernel, container runtime, and K8s version. Confirms all 5 nodes are `Ready` on `v1.35.8`.

```bash
kubectl get pods -A
```
> Lists every pod across all namespaces. Confirms no failing pods before we start. **Verify there is NO kube-proxy pod** — if there is, this cluster isn't in the state you think it is.

```bash
kubectl version
```
> Confirms client (bastion kubectl) and server (apiserver) versions. They should match pre-upgrade.

```bash
kubectl get deploy -A
kubectl get ds -A
kubectl get pdb -A
```
> Lists Deployments, DaemonSets, and PodDisruptionBudgets. **PDBs matter** — they govern how fast drains can proceed. In this lab, there are none.

```bash
helm list -n kube-system
```
> Confirms Cilium is Helm-managed and shows its chart version. Useful if you later need to upgrade the CNI.

**Expected baseline:**
- All 5 nodes `Ready` on `v1.35.8`
- All on `Ubuntu 24.04.4 LTS`
- All running `containerd://2.2.6`
- **No kube-proxy** in the pod list
- Client and server version match

### 3.2 From cp01

**SSH to cp01 first:**

```bash
ssh ubuntu@cp01.faekcorp.lab
```

```bash
containerd --version
```
> Confirms CRI version. Expected: `containerd github.com/containerd/containerd 2.2.6`.

```bash
kubelet --version
```
> Confirms kubelet version. Expected: `Kubernetes v1.35.8`.

```bash
kubeadm version
```
> Confirms kubeadm version. Expected: `v1.35.8`. This is the version we're about to replace.

### 3.3 etcd Cluster Verification

**Why:** etcd is the most critical component. If quorum is compromised *before* the upgrade, you must fix it first. Also, this defines your upgrade constraint: **3 members → quorum = 2 → you can never upgrade two control planes simultaneously.**

```bash
sudo etcdctl \
  --endpoints=https://10.70.21.6:2379,https://10.70.31.209:2379,https://10.70.41.250:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```
> Queries all three etcd endpoints directly using the etcd server cert. Shows version, leader, term, and raft index for each. **All three must be on the same version, same term, same raft index, with exactly one leader.**

**Expected:**
- All 3 members on 3.6.6
- Same term, same raft index, same applied index
- One member marked as leader (`true` in the Leader column)
- No errors

**Also verify member list:**

```bash
sudo etcdctl ... member list -w table
```
> Lists the etcd cluster members. All should be `started`, none should be `learner` or unhealthy.

**Quorum safety analysis:** 3 members → quorum = 2. We can lose exactly one at a time. **Constraint: never upgrade two control planes simultaneously.**

### 3.4 kubeadm-config Inspection

**Why:** This ConfigMap is the authoritative cluster configuration. Understanding it before you start prevents surprises.

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml
```
> Displays the full cluster configuration stored in the `kubeadm-config` ConfigMap. This is what `kubeadm upgrade` reads.

**Key findings to verify:**

1. `controlPlaneEndpoint: haproxy.faekcorp.lab:6443` — HA endpoint correctly configured
2. `kubernetesVersion: v1.35.8` — matches reality
3. `apiVersion: kubeadm.k8s.io/v1beta4` — modern config format
4. **`proxy: {}`** — empty. This does NOT mean "skip kube-proxy." It means "use default kube-proxy config." **This is the trap that caused our incident.**
5. **No `skipPhases`** anywhere

> **Important:** Finding #4 caused the main incident in this upgrade. The empty `proxy: {}` tells kubeadm "install kube-proxy as normal" — but this cluster doesn't have kube-proxy. We must prevent reinstallation. See [Section 7](#7-incident-response-kube-proxy-reinstall).

### 3.5 Capacity & PDB Review

**Why:** Draining a node evicts its workloads. You need to know there's room to absorb them.

```bash
kubectl describe nodes | grep -A5 -i allocated
```
> Shows CPU/memory requests and limits per node. Confirms workers have headroom to absorb evicted pods.

```bash
kubectl get pdb -A
```
> Lists PodDisruptionBudgets. If any exist, drains will respect them and may be slow. In this lab: none.

```bash
kubectl describe node cp01.faekcorp.lab | grep -A3 Taints
```
> Confirms control-plane taints. Expected: `node-role.kubernetes.io/control-plane:NoSchedule`. This means workloads won't land on CPs during drains — good, but CoreDNS tolerates the taint, so CoreDNS *can* run on CPs.

**Check:**
- Workers have headroom to absorb evicted pods
- CPs are `NoSchedule`-tainted (workloads won't land there during drains)
- PDBs — in this lab: none (drains will be fast, but no workload protection)

### 3.6 Phase 0 Sign-Off Checklist

**Why:** Do not proceed to Phase 1 until every row is green (or explicitly acknowledged).

| Check | Status | Notes |
|-------|:---:|-------|
| etcd cluster health | OK | All 3 members started, same version |
| etcd quorum safety (3 members, quorum=2) | OK | One CP at a time, never two |
| kubeadm-config present, correct | OK | But `proxy: {}` — see trap below |
| Node capacity headroom | OK | Workers nearly empty |
| cgroup v2 | OK | Required for modern kubelet |
| No PDBs | WARN | Expected for this lab |
| `skipPhases` for kube-proxy | MISSING | **Not present — must remember to skip manually** |
| CP taints present | OK | Standard control-plane taint |

---

## 4. Critical Concept: apply vs node

**Why:** This is the single most important concept in a kubeadm HA upgrade. Getting it wrong causes cluster-wide inconsistency.

### `kubeadm upgrade apply`

Run on **exactly one** control-plane node (the "authority"):
- **Writes cluster-wide state**: `kubeadm-config` ConfigMap, `kubelet-config` ConfigMap, addon deployments (CoreDNS, kube-proxy), bootstrap-token RBAC
- **Then** upgrades local static pod manifests
- **Only one CP should ever run `apply`**

### `kubeadm upgrade node`

Run on **every other** control-plane node, and on every worker:
- **Reads** the target version from `kubeadm-config`
- **Only** upgrades local static pod manifests (or, on workers, just kubelet config)
- **Does NOT touch** addons, RBAC, or cluster-wide ConfigMaps
- **Does NOT accept `--skip-phases`** — this is critical

### The Deferred Addon Phase — The Trap

**Why this matters:** In HA clusters, kubeadm **defers** the addon phase (CoreDNS, kube-proxy) until **all control planes are on the new version**. The output of cp01's `apply` shows:

```
[upgrade/addon] Skipping upgrade of addons because control plane instances [cp02, cp03] have not been upgraded
```

The deferred addon phase runs once, on the **last** CP's `upgrade node`. But `upgrade node`:
- Doesn't accept `--skip-phases`
- Has no memory of the flag passed to cp01's `apply`

**Result:** the deferred addon phase reinstalls kube-proxy on the last CP — with no way to prevent it via CLI. **The ConfigMap must be the authority.**

**Visual:**

```
              apply on cp01                    upgrade node on cp02, cp03
              ─────────────                    ──────────────────────────
Cluster state:
  ├─ kubeadm-config       ← WRITE              ← read only
  ├─ kubelet-config       ← WRITE              ← read only
  ├─ CoreDNS Deployment   ← WRITE              ← no action
  ├─ RBAC                 ← WRITE              ← no action
  └─ bootstrap tokens     ← refresh            ← no action

Local state:
  ├─ static pod manifests ← WRITE              ← WRITE
  └─ kubelet config       ← WRITE              ← WRITE
```

One authority. Many followers. If all CPs ran `apply`, they would race to update the same ConfigMaps and addons.

---

## 5. Phase 1 — Upgrade First Control-Plane (cp01)

**Why:** The first control-plane performs the **cluster-wide** upgrade of API objects and component manifests using `kubeadm upgrade apply`. Everything else follows.

> **Do not upgrade cp02 or cp03 until cp01 is fully done and verified.** Quorum requires 2 of 3 members. cp01's etcd will restart during this phase — cp02 and cp03 must be healthy to maintain quorum.

### Step 1 — SSH to cp01

```bash
ssh ubuntu@cp01.faekcorp.lab
```
> Connects to the first control-plane. All commands in Phase 1 run here unless stated otherwise.

### Step 2 — Baseline Capture

```bash
kubeadm version -o short
```
> Prints just the version string (e.g., `v1.35.8`). Confirms kubeadm is still the old version before we change it.

```bash
kubelet --version
```
> Confirms kubelet version. Expected: `Kubernetes v1.35.8`.

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```
> Shows which Kubernetes APT repo this node is configured to use. Expected: the `v1.35` repo.

```bash
apt-mark showhold
```
> Lists packages that are "held" (prevented from being auto-upgraded). Expected: `containerd.io`, `kubeadm`, `kubelet`. **These holds protect the cluster from accidental version drift.**

### Step 3 — Update APT Repo

**Why:** The APT repo URL contains the Kubernetes minor version. We must point it at `v1.36` before we can install v1.36 packages.

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
```
> Replaces `v1.35` with `v1.36` in the repo URL. **Note the `|` sed delimiter** instead of `/` — the repo URL contains `/`, and using `/` would require escaping each one. This is a common junior mistake.

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```
> Verifies the change. Expected: `https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /`.

### Step 4 — Refresh Package Index

```bash
sudo apt-get update
```
> Downloads the package lists from all configured repos, including the new v1.36 repo. Without this, `apt-cache madison` won't see the new versions.

**Expected:** You should see a line referencing `.../v1.36/deb InRelease`.

### Step 5 — Confirm Target Version Exists

**Why:** Never assume the package you want is available. Verify before pinning.

```bash
apt-cache madison kubeadm | grep 1.36.4
```
> Searches the apt cache for kubeadm 1.36.4. Expected output:
> ```
> kubeadm | 1.36.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.36/deb  Packages
> ```

```bash
apt-cache madison kubeadm | grep 1.36
```
> Shows the full range of 1.36 patches available. Confirms both our target (1.36.4) and the newest (1.36.5) exist. **We deliberately pick 1.36.4, not 1.36.5.**

### Step 6–8 — Install kubeadm 1.36.4

**Why:** `kubeadm` is the "decider" — it's the tool that orchestrates the upgrade. We upgrade it first, but leave kubelet and control-plane components alone for now.

```bash
sudo apt-mark unhold kubeadm
```
> Removes the hold on kubeadm so apt will allow the upgrade. **Note: we only unhold what we're about to change.**

```bash
sudo apt-get install -y kubeadm=1.36.4-1.1
```
> Installs kubeadm 1.36.4, replacing 1.35.8. **Note the `=1.36.4-1.1` — the exact version, including the Debian revision suffix. Without it, apt would install the newest 1.36.x.**

```bash
sudo apt-mark hold kubeadm
```
> Re-applies the hold. This prevents future `apt-get upgrade` runs from silently moving kubeadm.

**Note:** kubelet is still 1.35.8. We only changed the *decider*.

### Step 9 — Verify

```bash
kubeadm version -o short
```
> Expected: `v1.36.4`. Confirms the new kubeadm is installed.

### Step 10 — Pre-Pull Container Images

**Why:** Downloading control-plane images takes time. Doing it now means the upgrade won't stall waiting on the network — the images are already on disk.

```bash
sudo kubeadm config images pull --kubernetes-version v1.36.4
```
> Downloads all images kubeadm needs for 1.36.4. Expected pulls:
> - kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy → v1.36.4
> - coredns → v1.14.2
> - pause → 3.10.2
> - etcd → 3.6.8-0

**Verify images on disk:**

```bash
sudo crictl images
```
> Lists all container images cached on this node. You should see **both** v1.35.8 and v1.36.4 versions of each control-plane image, plus the new CoreDNS and etcd.

### Step 11 — Dry-Run: `kubeadm upgrade plan`

**Why:** This is a **read-only** dry run. It tells you exactly what will be upgraded, verifies the target version exists, and surfaces warnings. **Never skip this.**

```bash
sudo kubeadm upgrade plan v1.36.4
```
> Prints the full upgrade plan. Does not modify anything.

**What to check in the plan:**

1. Warnings about kube-proxy ConfigMap — expected, since we deleted it
2. Cluster version 1.35.8, target 1.36.4 — correct
3. Manual kubelet upgrade required on all 5 nodes — expected
4. All three control-plane components listed — apiserver, CM, scheduler
5. **`kube-proxy` in the list** — this is what we must skip
6. CoreDNS 1.13.1 → 1.14.2 — addon upgrade
7. etcd 3.6.6 → 3.6.8 — same minor, patch upgrade
8. `MANUAL UPGRADE REQUIRED: no` for both config rows — no config migration needed
9. Recommended command has no `--skip-phases` — we add it manually

**Verdict:** plan clean. Proceed.

### Step 12 — Apply

**Why:** This is the real thing. `kubeadm upgrade apply` writes cluster-wide state and upgrades cp01's static pod manifests. It's also the point of no return for cp01's control plane.

```bash
sudo kubeadm upgrade apply v1.36.4 --skip-phases=addon/kube-proxy
```
> Runs the upgrade, skipping the kube-proxy addon phase. When prompted, confirm with `y`.

> **Important:** The `--skip-phases=addon/kube-proxy` flag **only affects this invocation**. It does NOT persist in cluster state. When the *last* CP runs `upgrade node`, kubeadm will re-run the deferred addon phase — and will reinstall kube-proxy unless the ConfigMap tells it not to. See Section 7.

**What the output tells us:**

1. **etcd upgraded first** — new manifest written, certs renewed, pod restarted
2. **apiserver next** — certs renewed (server, kubelet-client, front-proxy, etcd-client)
3. **controller-manager and scheduler after** — in order
4. **Each component waited for health before moving on** — this is why the whole sequence is safe
5. **Cluster-wide state updated** — `kubeadm-config`, `kubelet-config`, RBAC rules
6. **`[addons]` line NOT present** — our `--skip-phases=addon/kube-proxy` worked for this run
7. **`[upgrade/addon] Skipping upgrade of addons because cp02/cp03 not upgraded`** — critical line. kubeadm is deferring addon application to the last CP's `upgrade node`.

**Watch from bastion (second terminal):**

```bash
kubectl get pod -n kube-system -w
```
> Live-watches kube-system pods. Each static pod restarts in sequence. The watch survives the transitions because HAProxy routes the watch request to cp02/cp03 during cp01's apiserver restart. **This is a good smoke test of HAProxy HA.**

**Expected sequence:**
- etcd pod → Terminating → Pending → Running
- apiserver pod → 0/1 → Running
- controller-manager pod → restarts
- scheduler pod → restarts

Everything comes back green.

### Step 13 — Upgrade kubelet on cp01

**Why:** kubeadm upgraded the *control plane* but not the *kubelet*. kubelet must match the apiserver minor (it's allowed to lag, but we want it current).

```bash
sudo apt-mark unhold kubelet
```
> Removes the hold so we can upgrade kubelet.

```bash
sudo apt-get install -y kubelet=1.36.4-1.1
```
> Installs kubelet 1.36.4. **Note:** the `.deb` post-install script **automatically restarts kubelet**.

```bash
sudo apt-mark hold kubelet
```
> Re-applies the hold.

```bash
sudo systemctl daemon-reload
```
> Tells systemd to re-read unit files (kubelet's unit may have changed).

```bash
sudo systemctl restart kubelet
```
> Restarts kubelet explicitly. **This causes all static pods to restart again.**

```bash
kubelet --version
sudo systemctl is-active kubelet.service
```
> Expected: `Kubernetes v1.36.4`, `active`.

> **Known behavior:** The `systemctl restart kubelet` causes **all static pods to restart again**. This is expected with containerd 2.x — static pod containers are re-created even if manifests haven't changed. Not a problem; pods come back healthy. **Do not panic.**

### Step 14 — Verify cp01

**From the bastion:**

```bash
kubectl get nodes
```
> Expected:
> ```
> cp01.faekcorp.lab      Ready    control-plane   7d23h   v1.36.4   ← UPGRADED
> cp02.faekcorp.lab      Ready    control-plane   7d20h   v1.35.8
> cp03.faekcorp.lab      Ready    control-plane   7d20h   v1.35.8
> worker1.faekcorp.lab   Ready    worker          6d22h   v1.35.8
> worker2.faekcorp.lab   Ready    worker          6d22h   v1.35.8
> ```

**Phase 1 complete.**

> **Checkpoint:** Before moving to Phase 2, confirm:
> - cp01 is `Ready` on v1.36.4
> - All control-plane pods on cp01 are `Running`
> - etcd cluster still has 3 healthy members
> - No new `Warning` events in `kubectl get events -A`

---

## 6. Phase 2 — Upgrade Remaining Control-Planes (cp02, cp03)

**Why:** Secondary control-planes use `kubeadm upgrade node` instead of `apply`. They don't write cluster state — they only upgrade their own static pod manifests.

> **Do one CP at a time.** Upgrading two simultaneously breaks etcd quorum.

### 6.1 cp02 — Second Control Plane

#### Step 1 — Repo change and kubeadm install

```bash
ssh ubuntu@cp02.faekcorp.lab
```
> Connects to cp02.

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```
> Same repo change and refresh as cp01.

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short
```
> Expected: `v1.36.4`.

#### Step 2 — Upgrade the node

```bash
sudo kubeadm upgrade node
```
> Upgrades cp02's static pod manifests. Reads the target version from `kubeadm-config`. **No `--skip-phases` accepted.**

**Key output lines:**

- No `[upgrade/addon] Applied essential addon` — addons deferred
- **Two identical skip lines** — `Skipping upgrade of addons because control plane instances [cp03.faekcorp.lab] have not been upgraded`
- Everything else upgraded successfully

#### Step 3 — Upgrade kubelet

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubelet --version
```
> Expected: `Kubernetes v1.36.4`.

#### Step 4 — Verify etcd cluster

**Why:** etcd leadership may have moved when cp02 restarted. Verify quorum is still intact before touching cp03.

```bash
sudo etcdctl \
  --endpoints=https://10.70.21.6:2379,https://10.70.31.209:2379,https://10.70.41.250:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

**Expected:**
- cp01 and cp02 on 3.6.8
- cp03 still on 3.6.6
- Leadership may have moved (expected when etcd restarts)
- All three in sync (same raft index/applied index)

#### Step 5 — Verify from bastion

```bash
kubectl get nodes
```
> Expected: 2 of 3 CPs on 1.36.4. **Quorum preserved.**

> **Checkpoint:** cp01 and cp02 on v1.36.4. cp03 still on v1.35.8. etcd healthy. Proceed to cp03.

### 6.2 cp03 — Third Control Plane (The Moment of Truth)

> **This is where kube-proxy gets reinstalled if you don't handle the ConfigMap.** Read Step 2 carefully.

#### Step 1 — Repo change and kubeadm install

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short
```
> Expected: `v1.36.4`.

#### Step 2 — Upgrade the node

```bash
sudo kubeadm upgrade node
```

**Important — watch for these two lines at the end:**

```
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy              ← THE PROBLEM
```

1. **`[addons] Applied essential addon: CoreDNS`** — expected. CoreDNS upgrades when the last CP completes.
2. **`[addons] Applied essential addon: kube-proxy`** — unexpected. **kube-proxy reinstalled.** This cluster does not use kube-proxy. Its presence will break CoreDNS.

**This is where the incident begins.** Proceed immediately to [Section 7](#7-incident-response-kube-proxy-reinstall).

#### Step 3 — Upgrade kubelet

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Step 4 — Verify nodes

**From bastion:**

```bash
kubectl get nodes
```
> Expected: All 3 CPs on 1.36.4.

#### Step 5 — Check for the kube-proxy problem

**From bastion:**

```bash
kubectl -n kube-system get pods | grep -E 'coredns|kube-proxy'
```

**If you see:**
- 5 kube-proxy pods running
- 2 new CoreDNS pods stuck at `0/1`

**Go to [Section 7](#7-incident-response-kube-proxy-reinstall) immediately. Do NOT proceed to worker upgrades with kube-proxy present.**

---

## 7. Incident Response: kube-proxy Reinstall

**Why this section exists:** This is the incident we hit. It will happen again on the next upgrade unless the ConfigMap is fixed. Read it, understand it, and fix it before it bites you.

### 7.1 Timeline (What Happened)

| Time | Event |
|------|-------|
| ~16:48 | cp03 runs `kubeadm upgrade node` |
| ~16:48 | `[addons] Applied essential addon: CoreDNS` |
| ~16:48 | `[addons] Applied essential addon: kube-proxy` — **kube-proxy reinstalled** |
| ~16:49 | kube-proxy DS created on all 5 nodes |
| ~16:50 | CoreDNS pods begin rolling update to 1.14.2 |
| ~16:52 | New CoreDNS pods on worker1 and worker2 come up `0/1` |
| ~16:53 | First readiness probe failures: `503` |
| ~17:00 | Diagnosis begins |
| ~17:05 | kube-proxy DS deleted |
| ~17:05 | kube-proxy ConfigMap deleted |
| ~17:06 | Discovery: iptables still has KUBE-* rules |
| ~17:10 | iptables cleanup begins |
| ~17:15 | All nodes cleaned |
| ~17:16 | Stuck CoreDNS pods deleted and recreated |
| ~17:18 | Both CoreDNS pods `1/1` Ready |

### 7.2 Immediate Symptoms

**Symptom 1 — kube-proxy pods running:**

```bash
kubectl -n kube-system get pods | grep kube-proxy
```
> Expected (bad): 5 kube-proxy pods, one per node.

**Symptom 2 — new CoreDNS pods not Ready:**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```
> Expected (bad): new CoreDNS pods `0/1`, old CoreDNS pod still `1/1` (keeping DNS alive).

**Why the cluster didn't fully lose DNS:** The old CoreDNS pod on cp03 (1.13.1) was still running and serving DNS. **Never delete the last healthy replica of a critical service while its replacement is failing readiness.**

### 7.3 CoreDNS Fallout — Diagnostics

**Event on the stuck pod:**

```bash
kubectl -n kube-system describe pod <coredns-pod> | tail -10
```
> Expected: `Readiness probe failed: HTTP probe failed with statuscode: 503`.

**Pod logs:**

```bash
kubectl -n kube-system logs <coredns-pod>
```

**Key log lines:**
```
[INFO] plugin/kubernetes: waiting for Kubernetes API before starting server
[INFO] plugin/ready: Plugins not ready: "kubernetes"
[WARNING] plugin/kubernetes: starting server with unsynced Kubernetes API
```

**Reading the logs:**
- CoreDNS started its DNS server on port 53
- But its `kubernetes` plugin **never synced with the API**
- The `plugin/ready` endpoint kept returning "not ready" → readiness probe 503
- The warning `starting server with unsynced Kubernetes API` confirms the plugin gave up waiting

**The `kubernetes` plugin needs to reach `https://kubernetes.default.svc.cluster.local` (ClusterIP 10.96.0.1:443).** If it can't reach the API, it can't sync, and it stays not Ready.

### 7.4 iptables Discovery

**Why:** Deleting the kube-proxy DaemonSet removes the pods but **does NOT remove the iptables rules they wrote**. Those rules persist and interfere with Cilium's eBPF service path.

```bash
sudo iptables-save | grep -c KUBE
```
> Counts iptables rules containing "KUBE". Expected (bad): a non-zero count (e.g., 94).

```bash
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
```
> Lists the KUBE-SERVICES chain in the nat table with line numbers. **The smoking gun:** rule for `10.96.0.1:443` being intercepted by kube-proxy's `KUBE-SVC-NPX46M4PTMTKRN6Y` chain, **before** Cilium's eBPF could handle it.

**Conclusion:** kube-proxy's iptables rules were intercepting ClusterIP traffic. The stale DNAT rules couldn't reach the API correctly, so new CoreDNS pods couldn't sync.

### 7.5 iptables Cleanup

**On every node (cp01, cp02, cp03, worker1, worker2):**

```bash
sudo iptables-save | grep -v KUBE | sudo iptables-restore
```
> **What this does:**
> 1. Saves all current iptables rules
> 2. Filters out any line containing `KUBE`
> 3. Restores the filtered ruleset

**Verify on each node:**

```bash
sudo iptables-save | grep -c KUBE
```
> Expected: `0`.

> **Note on KUBE-KUBELET-CANARY:** the kubelet sometimes regenerates this single canary chain. If a small non-zero count remains, check which chains:
> ```bash
> sudo iptables-save | grep KUBE
> ```
> If the only remaining entry is `KUBE-KUBELET-CANARY`, that's fine — it's kubelet's own, not kube-proxy's.

### 7.6 CoreDNS Restart

**With iptables cleaned, restart the stuck CoreDNS pods:**

```bash
kubectl -n kube-system delete pod <coredns-pod-1> <coredns-pod-2>
```
> Deletes the stuck pods. The Deployment recreates them.

**Verify:**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```
> Expected: Both new pods come up `1/1`.

**Why the restart worked this time:** the new pod's kubernetes plugin could now reach `10.96.0.1:443` via Cilium's eBPF path (no more kube-proxy iptables interference). The plugin synced with the API. Readiness probe passed.

### 7.7 kube-proxy Removal

```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
```
> Deletes the reinstalled kube-proxy DaemonSet and ConfigMap.

**Verify:**

```bash
kubectl get ds -A
kubectl get cm -n kube-system | grep kube-proxy
```
> Expected: No kube-proxy in either output.

### 7.8 ConfigMap Fix — The Correct Field

**Why:** We need to tell kubeadm "this cluster does not use kube-proxy" so the deferred addon phase skips it on future upgrades.

**First attempt (`skipPhases`) was WRONG:**

```yaml
    proxy: {}
    scheduler: {}
    skipPhases:
    - addon/kube-proxy
```

**Why it failed:**
- `skipPhases` **is not a valid field** in `ClusterConfiguration`
- It belongs in `InitConfiguration` or `JoinConfiguration`
- The API server accepted the edit (it stores strings, doesn't validate embedded YAML), but kubeadm rejected it at parse time with:
  ```
  error unmarshaling configuration
  schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta4", Kind:"ClusterConfigura...
  ```

**Second attempt (`proxy.disabled: true`) is RIGHT:**

```bash
kubectl -n kube-system edit cm kubeadm-config
```
> Opens the ConfigMap in your default editor. Use `kubectl get cm kubeadm-config -o yaml > /tmp/backup.yaml` first as a safety net.

**Change:**

```yaml
    proxy: {}
    scheduler: {}
    skipPhases:
    - addon/kube-proxy
```

**To:**

```yaml
    proxy:
      disabled: true
    scheduler: {}
```

**Verify:**

```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | tail -8
```
> Prints the last 8 lines of the ClusterConfiguration. Expected:
> ```
> kubernetesVersion: v1.36.4
> networking:
>   dnsDomain: cluster.local
>   podSubnet: 10.244.0.0/16
>   serviceSubnet: 10.96.0.0/12
> proxy:
>   disabled: true
> scheduler: {}
> ```

**No `skipPhases`.** No unmarshal warning will be produced.

> **Known kubeadm bug (issue #3137):** `proxy.disabled: true` may be parsed by the plan phase but not honored by `apply`. So it's semantically correct but not yet reliable. **Always verify post-upgrade.**

### 7.9 Verification on Worker

**Re-run `kubeadm upgrade node` on a worker to confirm the ConfigMap fix:**

```bash
sudo kubeadm upgrade node
```

**Expected output:**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade/preflight] Running pre-flight checks
[upgrade/preflight] Skipping prepull. Not a control plane node.
[upgrade/control-plane] Skipping phase. Not a control plane node.
[upgrade/kubeconfig] Skipping phase. Not a control plane node.
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
[upgrade/addon] Skipping the addon/coredns phase. Not a control plane node.
[upgrade/addon] Skipping the addon/kube-proxy phase. Not a control plane node.
```

**The unmarshal error is gone.** The remaining warning about RBAC is benign:
- Workers don't have RBAC to read ConfigMaps in kube-system (correct security posture)
- There is no kube-proxy ConfigMap anyway
- kubeadm says "continuing without it" and proceeds

---

## 8. Phase 3 — Upgrade Workers (worker1, worker2)

**Why:** Workers run your workloads. Draining before upgrade evicts pods to other workers so applications keep running. **Upgrade one worker at a time.**

### 8.1 worker1

#### Step 1 — Cordon + Drain (from bastion)

```bash
kubectl cordon worker1.faekcorp.lab
```
> Marks worker1 as `SchedulingDisabled`. No new pods will be scheduled onto it, but existing pods keep running.

```bash
kubectl drain worker1.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```
> **What this does:**
> - `--ignore-daemonsets` — DaemonSet pods (Cilium, etc.) run on all nodes by design; don't try to evict them
> - `--delete-emptydir-data` — allows eviction of pods using emptyDir volumes (their data is ephemeral anyway)
> - `--timeout=10m` — gives evictions up to 10 minutes before giving up
> - Evicts regular workload pods; they reschedule to other available nodes
>
> **Expected:** Workloads migrate to worker2 in seconds.

> **Trap — bare Pods:** If you get `cannot delete Pods that declare no controller`, a debugging session left a bare Pod on this node. Delete it manually (`kubectl delete pod <name>`) and re-run the drain. **Do NOT use `--force`** unless you've investigated why the pod won't go away.

#### Step 2 — Upgrade on worker1

```bash
ssh ubuntu@worker1.faekcorp.lab
```
> Connects to worker1.

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```
> Same repo change and refresh as the CPs.

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short
```
> Expected: `v1.36.4`.

```bash
sudo kubeadm upgrade node
```
> **Note:** workers skip all control-plane and addon phases. Only the kubelet config is touched.

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubelet --version
sudo systemctl is-active kubelet.service
```
> Expected: `Kubernetes v1.36.4`, `active`.

#### Step 3 — Uncordon

```bash
kubectl uncordon worker1.faekcorp.lab
kubectl get nodes
```
> Expected: worker1 on v1.36.4. 4 of 5 nodes upgraded.

### 8.2 worker2

#### Step 1 — Cordon

```bash
kubectl cordon worker2.faekcorp.lab
```

#### Step 2 — Drain

```bash
kubectl drain worker2.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**If you get an error about bare Pods:**

```
error: unable to drain node "worker2.faekcorp.lab" due to error:
cannot delete Pods that declare no controller (use --force to override):
default/debug-pod
```

**Cause:** an earlier debugging session left a bare Pod on worker2. Bare Pods have no controller, so drain can't reschedule them.

**Fix:**

```bash
kubectl delete pod debug-pod
```

**Then re-run the drain:**

```bash
kubectl drain worker2.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

#### Step 3 — Upgrade on worker2

Same sequence as worker1:

```bash
ssh ubuntu@worker2.faekcorp.lab
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Step 4 — Uncordon

```bash
kubectl uncordon worker2.faekcorp.lab
kubectl get nodes
```
> Expected: All 5 nodes on v1.36.4.

---

## 9. Phase 4 — Post-Upgrade Validation

**Why:** Never trust an upgrade until you've proven the cluster works. This section exercises every critical path: CNI, etcd, ClusterIP, DNS, API.

> **All `kubectl` commands in this section run from the BASTION.**

### 9.1 Cilium Health

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status
```
> Runs `cilium-dbg status` inside the cilium-agent container. Shows Cilium's view of the cluster.

**Verify:**
- `Kubernetes: Ok 1.36 (v1.36.4)` — Cilium recognizes the upgraded API version
- `KubeProxyReplacement: True` — eBPF service path active
- `Cilium: Ok 1.20.1` — CNI healthy
- `Cluster health: 5/5 reachable` — all 5 nodes in the Cilium mesh
- `Controller Status: 18/18 healthy` — all internal controllers converged
- `Modules Health: Stopped(0) Degraded(0) OK(76)` — no degraded modules

### 9.2 etcd Cluster

```bash
sudo etcdctl \
  --endpoints=https://10.70.21.6:2379,https://10.70.31.209:2379,https://10.70.41.250:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```
> Same command as Phase 0. **Compare to baseline.**

**Verify:**
- All three on 3.6.8
- Same term — consistent cluster state
- Same raft index and applied index — no lag
- No errors

### 9.3 ClusterIP Functional Test

**Why:** This proves the eBPF service path works end-to-end. If kube-proxy's iptables rules are still interfering, this test will fail.

```bash
kubectl create ns post-upgrade-test
kubectl -n post-upgrade-test create deploy after-upgrade --image=nginx --replicas=3
kubectl -n post-upgrade-test expose deploy after-upgrade --port 80
```
> Creates a test namespace, a 3-replica nginx Deployment, and a ClusterIP Service.

**Verify pods spread across workers:**

```bash
kubectl -n post-upgrade-test get pods -o wide
```
> Confirms the scheduler placed pods on multiple workers.

**HTTP test through ClusterIP:**

```bash
kubectl -n post-upgrade-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://after-upgrade.post-upgrade-test.svc.cluster.local | head -5
```
> Runs a one-off busybox pod that fetches the Service's ClusterIP via DNS name. **This exercises DNS → ClusterIP → eBPF service path → pod.**

**Expected:** nginx welcome page HTML. ClusterIP works end-to-end.

### 9.4 DNS Test

**Do NOT use busybox `nslookup`** — it does not expand search domains and will give a false NXDOMAIN.

**Correct test — with netshoot (full bind-utils):**

```bash
kubectl run debug-pod --image=nicolaka/netshoot:latest -- sleep 3600
kubectl exec -it debug-pod -- sh
```
> Launches a netshoot pod (has full DNS tools) and drops you into a shell.

**Inside:**

```bash
nslookup kubernetes.default
nslookup after-upgrade.post-upgrade-test
```
> Expected: Correct FQDN expansion and resolution.

> **Note:** `Got recursion not available from 10.96.0.10` warnings are **expected**. CoreDNS is authoritative for cluster zones, not recursive. The `Name:` and `Address:` lines are the correct answer.

### 9.5 API Readyz

```bash
kubectl get --raw='/readyz?verbose'
```
> Queries the apiserver's readiness endpoint directly. **All checks should pass.**

**Expected:** `readyz check passed`.

### 9.6 System Pods

```bash
kubectl get pods -A | grep -vE 'Running|Completed'
```
> Lists any pod NOT in Running or Completed state. **Expected: empty output.**

### 9.7 DaemonSets

```bash
kubectl get ds -A
```
> Expected:
> ```
> kube-system   cilium         5/5/5/5/5
> kube-system   cilium-envoy   5/5/5/5/5
> ```
> **No kube-proxy.**

### 9.8 kube-proxy Absence

```bash
kubectl get ds -n kube-system kube-proxy
kubectl get cm -n kube-system kube-proxy
```
> Expected:
> ```
> Error from server (NotFound): daemonsets.apps "kube-proxy" not found
> Error from server (NotFound): configmaps "kube-proxy" not found
> ```

---

## 10. Bastion kubectl Upgrade

**Why:** kubectl may be ±1 minor from the apiserver, but matching is best practice. The bastion is where all `kubectl` commands run, so its client version should match the new server.

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubectl
sudo apt-get install -y kubectl=1.36.4-1.1
sudo apt-mark hold kubectl
kubectl version
```
> Expected:
> ```
> Client Version: v1.36.4
> Kustomize Version: v5.8.1
> Server Version: v1.36.4
> ```

**Client and server matched.**

---

## 11. Controlled Reboots

**Why we reboot:** After a Kubernetes upgrade, a controlled reboot of each node is recommended for three reasons:

1. **Pending kernel updates** — apt may have installed a new kernel without rebooting to activate it
2. **kubelet restarts leave transient state** — static pods restarted multiple times; a clean reboot consolidates
3. **Kubelet config changes** — a reboot ensures kubelet starts with the new config from a cold baseline

### 11.1 Pre-Reboot Kernel Status

**Why:** Check which nodes actually need a reboot before you start.

```bash
# On each node
uname -r
```

| Node | Kernel before reboots | Needs reboot? |
|------|----------------------|:---:|
| cp01 | 7.0.0-1013-aws | Yes |
| cp02 | 7.0.0-1013-aws | Yes |
| cp03 | 7.0.0-1014-aws | already current |
| worker1 | 7.0.0-1013-aws | Yes |
| worker2 | 7.0.0-1013-aws | Yes |

### 11.2 Reboot Procedure (per node)

**From bastion:**

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
```
> Same cordon/drain as the upgrade. Evacuates workloads.

**Reboot — use `ssh -t` or delayed reboot:**

```bash
ssh -t ubuntu@<node> 'sudo reboot'
```
> **Why `-t`:** forces TTY allocation, which matters for `sudo`. Without it, some SSH configurations close the session before `sudo reboot` fully executes. **This bit us on cp01 — the first reboot silently didn't take.**

**Or, more reliable:**

```bash
ssh ubuntu@<node> 'sudo shutdown -r +1 "Kernel update reboot"'
```
> Schedules a reboot 1 minute in the future. The SSH session exits cleanly, then the reboot executes.

**Watch from another terminal:**

```bash
kubectl get nodes -w
```
> Expected: Node goes `NotReady,SchedulingDisabled` → `Ready,SchedulingDisabled` in ~60 seconds.

**Verify kernel:**

```bash
ssh ubuntu@<node> uname -r
```
> Expected: `7.0.0-1014-aws`.

**Uncordon:**

```bash
kubectl uncordon <node>
```

### 11.3 CoreDNS Race During CP Reboots

**What happened:** During cp02's reboot, a CoreDNS pod was evicted and rescheduled to worker2, which was still stabilizing after its own reboot. The pod came up `0/1` and stayed there.

**Diagnosis:** worker2's kubelet and/or Cilium agent weren't fully ready. The pod's kubernetes plugin couldn't sync with the API.

**The fix:** wait. The pod's readiness probe retries continuously. Within a couple of minutes, worker2 was fully back, and the pod either recovered on its own or was replaced.

> **Lesson:** During rolling reboots, pods can be scheduled onto nodes that aren't fully ready. The pod will appear `Running` but fail readiness until the node stabilizes. **Wait 2–3 minutes before intervening.** Do not delete the pod immediately — you may be deleting a pod that's about to recover.

### 11.4 cp01 Half-Reboot Incident

**First attempt:**

```bash
ssh ubuntu@cp01.faekcorp.lab
sudo reboot
```

**Problem:** the SSH session closed, and the bastion session came back too fast. Subsequent `uname -r` showed the old kernel — the reboot hadn't actually taken.

**Root cause:** the `reboot` command was interrupted by the SSH session closing before it could execute.

**Fix:** use `ssh -t`:

```bash
ssh -t ubuntu@cp01.faekcorp.lab 'sudo reboot'
```

**Or, more reliable — delayed reboot:**

```bash
ssh ubuntu@cp01.faekcorp.lab 'sudo shutdown -r +1 "Kernel update reboot"'
```

---

## 12. Kernel Update Procedure

**Why:** After reboots, cp01 was still on the old kernel while the other four nodes were current. This section explains how to diagnose and fix that.

### 12.1 When a Kernel Update Appears "Stuck"

After all reboots, cp01 was **still on kernel 7.0.0-1013-aws** while the other four nodes showed `7.0.0-1014-aws`.

### 12.2 Diagnosis

**Check installed kernels:**

```bash
dpkg -l | grep linux-image
```
> Lists every installed kernel package. **Key finding: `linux-image-7.0.0-1014-aws` is NOT in the installed list on cp01.**

> **Reboots can't fix this.** The 1014 kernel package was never installed on cp01. No number of reboots will activate a kernel that isn't on disk.

### 12.3 Why This Happened

The other four nodes had `linux-image-aws` pointing at the new 1014 kernel. cp01 did not. Most likely:
- An `apt-get upgrade` on the other nodes pulled in the new metapackage → 1014 kernel
- On cp01, the same command either didn't run or was interrupted
- The metapackage on cp01 stayed at 1013

### 12.4 The Fix

**Check what's available:**

```bash
apt-cache policy linux-image-aws linux-image-7.0.0-1014-aws
```
> Expected:
> ```
> linux-image-aws:
>   Installed: 7.0.0-1013.13~24.04.1
>   Candidate: 7.0.0-1014.14~24.04.1        ← newer version is available
> 
> linux-image-7.0.0-1014-aws:
>   Installed: (none)                        ← not installed
>   Candidate: 7.0.0-1014.14~24.04.1        ← available to install
> ```

**Install the kernel:**

```bash
sudo apt-get install -y linux-image-aws linux-headers-aws
```
> Installs the new kernel metapackage and headers. This pulls in `linux-image-7.0.0-1014-aws`.

**Verify the symlink:**

```bash
ls -la /boot/vmlinuz /boot/vmlinuz.old
```
> Expected:
> ```
> /boot/vmlinuz -> vmlinuz-7.0.0-1014-aws
> /boot/vmlinuz.old -> vmlinuz-7.0.0-1013-aws
> ```

**Reboot:**

```bash
sudo reboot
```

**Verify:**

```bash
uname -r
```
> Expected: `7.0.0-1014-aws`.

### 12.5 Lesson

**When a kernel update appears "stuck", check what's actually installed** (`dpkg -l | grep linux-image`) before assuming the reboot isn't taking. An uninstalled kernel will never boot, no matter how many reboots you do.

---

## 13. Final State Checklist

**Why:** Before you call the upgrade done, verify every row. This is your sign-off.

### 13.1 Nodes

```
NAME                   STATUS   ROLES           VERSION   KERNEL
cp01.faekcorp.lab      Ready    control-plane   v1.36.4   7.0.0-1014-aws
cp02.faekcorp.lab      Ready    control-plane   v1.36.4   7.0.0-1014-aws
cp03.faekcorp.lab      Ready    control-plane   v1.36.4   7.0.0-1014-aws
worker1.faekcorp.lab   Ready    worker          v1.36.4   7.0.0-1014-aws
worker2.faekcorp.lab   Ready    worker          v1.36.4   7.0.0-1014-aws
```

**Every node:**
- Ready
- Kubernetes v1.36.4
- Kernel 7.0.0-1014-aws
- containerd 2.2.6

### 13.2 System Pods

```bash
kubectl get pods -A | grep -vE 'Running|Completed'
```
> Expected: Empty output.

### 13.3 CoreDNS

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```
> Expected: Both pods `1/1` Ready. Zero restarts since last rollout.

### 13.4 DaemonSets

```bash
kubectl get ds -A
```
> Expected:
> ```
> kube-system   cilium         5/5/5/5/5
> kube-system   cilium-envoy   5/5/5/5/5
> ```
> **No kube-proxy.**

### 13.5 kube-proxy Absence

```bash
kubectl get ds -n kube-system kube-proxy
kubectl get cm -n kube-system kube-proxy
```
> Expected: `NotFound` for both.

### 13.6 ConfigMap

```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | tail -8
```
> Expected:
> ```
> kubernetesVersion: v1.36.4
> networking:
>   dnsDomain: cluster.local
>   podSubnet: 10.244.0.0/16
>   serviceSubnet: 10.96.0.0/12
> proxy:
>   disabled: true
> scheduler: {}
> ```

### 13.7 Cilium

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status
```
> Expected:
> ```
> Kubernetes:              Ok         1.36 (v1.36.4)
> KubeProxyReplacement:    True
> Cilium:                  Ok   1.20.1
> Routing:                 Network: Tunnel [geneve]
> Cluster health:          5/5 reachable
> Controller Status:       18/18 healthy
> Modules Health:          Stopped(0) Degraded(0) OK(76)
> ```

### 13.8 etcd

```bash
sudo etcdctl ... endpoint status -w table
```
> Expected: All 3 members on 3.6.8, same term, same raft index, no errors.

### 13.9 What Changed

| Item | Before | After |
|------|--------|-------|
| Kubernetes version | v1.35.8 | v1.36.4 |
| kubelet | v1.35.8 | v1.36.4 |
| kubeadm | v1.35.8 | v1.36.4 |
| kubectl (bastion) | v1.35.8 | v1.36.4 |
| etcd | 3.6.6 | 3.6.8 |
| CoreDNS | 1.13.1 | 1.14.2 |
| `kubeadm-config` `proxy` field | `{}` | `disabled: true` |
| Kernel (all 5 nodes) | 7.0.0-1013-aws | 7.0.0-1014-aws |
| iptables KUBE-* rules | Present on all nodes | Removed |

### 13.10 What Stayed the Same

| Item | Version |
|------|---------|
| Cilium | 1.20.1 |
| containerd | 2.2.6 |
| HAProxy | (unchanged) |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |
| API endpoint | haproxy.faekcorp.lab:6443 |
| kube-proxy | Removed (still) |

---

## 14. Lessons Learned / What To Do Differently

**Why:** The next upgrade will be smoother if we internalize these.

### 14.1 Lessons Learned

1. **`--skip-phases` on `apply` is per-invocation.** It doesn't persist in cluster config, and the deferred addon phase on the last CP has no memory of it.

2. **Deleting a DaemonSet does not remove its iptables rules.** Clean them explicitly on every node.

3. **`proxy.disabled: true` in `ClusterConfiguration` is the correct field**, but kubeadm has a known bug where it's not honored during `upgrade apply`. Never rely on it alone.

4. **After every control-plane upgrade, verify kube-proxy is gone** (DS and ConfigMap) and clean iptables on every node.

5. **`skipPhases` is not a `ClusterConfiguration` field.** It belongs in `InitConfiguration` or `JoinConfiguration`. Putting it in `ClusterConfiguration` causes an unmarshal error.

6. **New pods can fail readiness when their ClusterIP path is broken.** The symptom is Running-but-not-Ready with a `503` readiness probe. CoreDNS's kubernetes plugin logs "waiting for Kubernetes API."

7. **Never delete the last healthy replica** of a critical service while its replacement is failing readiness.

8. **Different DNS tools behave differently.** Busybox's nslookup doesn't expand search domains. Use `nicolaka/netshoot` or test with FQDNs.

9. **Pods can be scheduled onto nodes that aren't fully ready after a reboot.** They'll appear Running but fail readiness for 1–3 minutes. Wait before investigating.

10. **Unmanaged pods (bare Pods) block drains.** Delete them manually rather than using `--force`.

11. **Reboots can be interrupted by SSH session termination.** Use `ssh -t` or `shutdown -r +1`.

12. **The etcd cluster tolerates one CP restart at a time.** Two simultaneous restarts break quorum.

13. **HAProxy routing is transparent during CP reboots.** The API stayed available throughout.

14. **kubelet restart causes static pods to cycle on that node.** Expected behavior with containerd 2.x.

15. **`kubeadm upgrade node` on workers skips control-plane and addon phases.** Correct behavior.

16. **When a kernel update appears stuck, check what's installed.** `dpkg -l | grep linux-image` before assuming the reboot isn't working.

### 14.2 What We Would Do Differently

1. **Patch the `kubeadm-config` ConfigMap with `proxy.disabled: true` BEFORE the upgrade**, not after.

2. **Add `skipPhases` to the `InitConfiguration`/`JoinConfiguration`**, not `ClusterConfiguration`.

3. **After cp03's `upgrade node`, immediately check for kube-proxy reinstallation.** Don't assume `--skip-phases` on cp01 was sufficient.

4. **Clean iptables proactively** after any kube-proxy removal. Don't wait for a symptom.

5. **Take a fresh etcd snapshot immediately before the upgrade** — even in a lab.

6. **Test DNS with FQDNs**, not short names.

7. **Reboot with `ssh -t`** or `shutdown -r +1`. Avoid the half-reboot problem.

8. **Check installed kernel packages** before spending time on reboots. If the package isn't there, no reboot will activate it.

9. **Document each step's expected output before executing.** This runbook was built as we went; next time we start from it.

---

## 15. Appendices

### Appendix A — etcd Snapshot Procedure (Pre-Upgrade)

**Why:** The etcd snapshot is your rollback plan. If the upgrade fails catastrophically, this is how you restore the cluster.

**Not executed during this upgrade** (lab environment, no new state since previous snapshot). **In production, this is non-negotiable.**

```bash
# Create backup directory
sudo mkdir -p /root/etcd-backups
```
> Creates a dedicated directory for backups. Keeps them organized and easy to find.

```bash
# Take the snapshot
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot.db
```
> Takes a point-in-time snapshot of the etcd database. **This is your rollback point.**

```bash
# Copy with timestamp
sudo cp /var/lib/etcd/snapshot.db \
  /root/etcd-backups/snapshot-pre-upgrade-$(date +%Y%m%d-%H%M%S).db
```
> Copies the snapshot to the backup directory with a timestamp. **Never overwrite old snapshots** — you may need to roll back further than the most recent one.

```bash
# Verify integrity
sudo etcdutl snapshot status \
  /root/etcd-backups/snapshot-pre-upgrade-*.db -w table
```
> Uses `etcdutl` (not `etcdctl`) to verify the snapshot is valid. Expected: valid hash, key count, size.

```bash
# Back up cluster identity
sudo tar -czf \
  /root/etcd-backups/kubernetes-config-pre-upgrade-$(date +%Y%m%d-%H%M%S).tar.gz \
  /etc/kubernetes
```
> Archives `/etc/kubernetes` — this contains the cluster's CAs, certs, and kubeconfigs. **Why both are needed:** the etcd snapshot has cluster *state*; the `/etc/kubernetes` tarball has cluster *identity* (CAs, certs). Both are required for full recovery.

### Appendix B — containerd Upgrade Procedure

**Why:** If containerd is below the minimum for your target K8s version, you must upgrade it *before* Kubernetes.

**Not executed during this upgrade** — containerd 2.2.6 satisfied the 1.36 minimum of `2.2.0+`.

**Reference — Kubernetes-to-containerd compatibility:**

| Kubernetes | containerd minimum |
|-----------|-------------------|
| 1.32 | 2.1.0+, 2.0.1+, 1.7.24+, 1.6.36+ |
| 1.33 | 2.1.0+, 2.0.4+, 1.7.24+, 1.6.36+ |
| 1.34 | 2.1.3+, 2.0.6+, 1.7.28+, 1.6.39+ |
| 1.35 | 2.2.0+, 2.1.5+, 1.7.28+ |
| 1.36 | 2.3.0+, 2.2.0+ |
| 1.37 | 2.4.0+, 2.3.0+ |

**When to upgrade:** before a Kubernetes upgrade if current containerd is below the target K8s version's minimum. Otherwise it can wait.

**How to upgrade containerd — per node:**

```bash
containerd --version
```
> Confirms current version.

```bash
sudo apt-mark unhold containerd.io
```
> Removes the hold so we can upgrade containerd.

```bash
apt-cache madison containerd.io
```
> Shows available versions.

```bash
sudo apt-get install -y containerd.io=<target-version>
```
> Installs the target version. **Pin the exact version.**

```bash
sudo apt-mark hold containerd.io
```
> Re-applies the hold.

```bash
sudo grep -E 'SystemdCgroup|sandbox_image' /etc/containerd/config.toml
```
> Verifies the containerd config was preserved. **Critical:** if `SystemdCgroup = true` is missing, kubelet won't work correctly.

```bash
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl is-active containerd
```
> Restarts containerd. **Warning:** this briefly interrupts the CRI on that node. Running pods keep running, but new container operations queue.

```bash
sudo ctr plugins ls | grep -E 'cri|grpc'
sudo crictl info
```
> Verifies the CRI plugin is loaded and responsive.

```bash
sudo systemctl status kubelet
```
> Confirms kubelet reconnected to containerd.

**Order for cluster-wide:**
1. One worker at a time (cordon, drain, upgrade, uncordon)
2. Then control planes one at a time
3. Verify cluster health between nodes

### Appendix C — CNI Compatibility Verification

**Why:** The CNI is the cluster's network fabric. Upgrading K8s without a compatible CNI can break all pod networking.

**Not executed as a separate step** — Cilium compatibility was verified during Phase 0 planning.

**Check current CNI version:**

```bash
helm list -n kube-system
```
> Shows Helm-managed releases. Confirms Cilium is Helm-managed and its chart version.

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium version
```
> Shows the running Cilium version.

```bash
kubectl get ds -n kube-system cilium -o yaml | grep -i image
```
> Shows the exact Cilium image tag.

**Reference — Cilium compatibility:**

| Cilium | Supported Kubernetes |
|--------|----------------------|
| 1.20 | >=1.33 <1.37 |
| 1.19 | >=1.32 <1.36 |
| 1.18 | >=1.30 <1.34 |

**If CNI needs upgrading:**
1. Upgrade CNI first, **before** the K8s upgrade
2. Verify the CNI still supports the *current* K8s version
3. Verify Cilium status: `KubeProxyReplacement: True`, cluster health OK
4. Then proceed with the Kubernetes upgrade
5. After K8s is upgraded, verify CNI status again

### Appendix D — Troubleshooting Command Reference

**Cluster state:**

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pods -A | grep -vE 'Running|Completed'
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
kubectl get --raw='/readyz?verbose' | tail -5
```

**etcd:**

```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

sudo etcdctl ... endpoint status -w table
sudo etcdctl ... endpoint health -w table
```

**Cilium:**

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status
kubectl get ciliumnodes
kubectl get ds -A
```

**CoreDNS:**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
kubectl -n kube-system describe pod <coredns-pod>
```

**kube-proxy / iptables:**

```bash
kubectl get ds -A | grep kube-proxy
kubectl get cm -n kube-system | grep kube-proxy
sudo iptables-save | grep -c KUBE
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
```

**kubeadm-config:**

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'
```

**Worker node:**

```bash
sudo systemctl status kubelet --no-pager
sudo systemctl status containerd --no-pager
sudo journalctl -u kubelet -n 50
sudo crictl ps -a
sudo crictl pods -a
```

### Appendix E — Complete Command Index

**Phase 0 — Pre-flight:**

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl version
kubectl get deploy -A
kubectl get ds -A
kubectl get pdb -A
sudo etcdctl --endpoints=... member list -w table
kubectl -n kube-system get cm kubeadm-config -o yaml
kubectl describe nodes | grep -A5 -i allocated
```

**Phase 1 — cp01:**

```bash
ssh ubuntu@cp01.faekcorp.lab
kubeadm version -o short
kubelet --version
cat /etc/apt/sources.list.d/kubernetes.list
apt-mark showhold
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm | grep 1.36.4
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short
sudo kubeadm config images pull --kubernetes-version v1.36.4
sudo kubeadm upgrade plan v1.36.4
sudo kubeadm upgrade apply v1.36.4 --skip-phases=addon/kube-proxy
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubelet --version
```

**Phase 2 — cp02, cp03:**

```bash
ssh ubuntu@cp02.faekcorp.lab     # or cp03
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**Incident response:**

```bash
# Diagnose CoreDNS
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system describe pod <coredns-pod> | tail -10
kubectl -n kube-system logs <coredns-pod>

# Check iptables
sudo iptables-save | grep -c KUBE
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers

# Clean iptables
sudo iptables-save | grep -v KUBE | sudo iptables-restore

# Restart CoreDNS
kubectl -n kube-system delete pod <coredns-pod-1> <coredns-pod-2>

# Remove kube-proxy
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy

# Fix ConfigMap
kubectl -n kube-system get cm kubeadm-config -o yaml > /tmp/backup.yaml
kubectl -n kube-system edit cm kubeadm-config
# Change: proxy: {} → proxy:\n  disabled: true
# Remove: skipPhases block

# Verify
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | tail -8
```

**Phase 3 — Workers:**

```bash
# From bastion
kubectl cordon <worker>
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data --timeout=10m

# On worker
ssh ubuntu@<worker>
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# From bastion
kubectl uncordon <worker>
```

**Phase 4 — Validation:**

```bash
kubectl version
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status
sudo etcdctl ... endpoint status -w table
kubectl create ns post-upgrade-test
kubectl -n post-upgrade-test create deploy after-upgrade --image=nginx --replicas=3
kubectl -n post-upgrade-test expose deploy after-upgrade --port 80
kubectl -n post-upgrade-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://after-upgrade.post-upgrade-test.svc.cluster.local | head -5
kubectl run debug-pod --image=nicolaka/netshoot:latest -- sleep 3600
kubectl exec -it debug-pod -- nslookup kubernetes.default.svc.cluster.local
kubectl get --raw='/readyz?verbose' | tail -5
```

**Bastion kubectl:**

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubectl
sudo apt-get install -y kubectl=1.36.4-1.1
sudo apt-mark hold kubectl
kubectl version
```

**Controlled reboot:**

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
ssh -t ubuntu@<node> 'sudo reboot'
# wait for Ready
ssh ubuntu@<node> uname -r
kubectl uncordon <node>
```

**Kernel update:**

```bash
sudo apt-get update
apt-cache policy linux-image-aws linux-image-7.0.0-1014-aws
apt-mark showhold
sudo apt-get install -y linux-image-aws linux-headers-aws
ls -la /boot/vmlinuz /boot/vmlinuz.old
sudo reboot
uname -r
```

---
