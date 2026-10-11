# Kubernetes HA Cluster Upgrade Runbook
## v1.35.8 → v1.36.4 — Complete Execution Record

**Cluster:** faekcorp-lab
**Date of execution:** 2026-10-11 / 2026-10-12
**Document version:** 1.0 (complete)
**Status:** ✅ Complete — all nodes on v1.36.4, kernel 7.0.0-1014-aws

---

# PART A — CONTEXT, DECISIONS, AND PHASE 0

---

## Table of Contents — Part A

1. Purpose and scope
2. What this document is and is not
3. The cluster and its architecture
4. Target version selection
5. Compatibility verification (containerd, Cilium, etcd)
6. What we chose NOT to upgrade
7. Version skew rules applied
8. Phase 0 — pre-flight verification
9. Phase 0 — node state capture
10. Phase 0 — etcd cluster verification
11. Phase 0 — kubeadm-config inspection
12. Phase 0 — capacity and PDB review
13. What Phase 0 told us

---

## 1. Purpose and scope

This runbook documents the **actual, executed** upgrade of the faekcorp-lab Kubernetes cluster from **v1.35.8** to **v1.36.4**. It is a record of what we did, why, what went wrong, and what we learned.

It covers:
- Pre-flight verification
- Control-plane upgrades (cp01, cp02, cp03)
- Worker upgrades (worker1, worker2)
- Two major incidents (kube-proxy reinstall, cp01 kernel)
- Post-upgrade validation
- Controlled reboots
- Appendices: procedures not executed but documented

This document is **specific to a cluster where kube-proxy has been removed and replaced by Cilium's eBPF implementation**. Standard kubeadm documentation assumes kube-proxy exists. This cluster does not. That difference caused the primary incident in this upgrade.

---

## 2. What this document is and is not

**It is:**
- A faithful record of an executed upgrade, including all failures
- A reference for the next time this cluster needs upgrading
- A document that captures the *why* behind every step

**It is not:**
- A generic Kubernetes upgrade tutorial
- A "best practices" guide
- Complete in every detail — some output was abbreviated where not meaningful

---

## 3. The cluster and its architecture

### Nodes

| Node | Role | Internal IP | OS | containerd | Pre-upgrade K8s |
|------|------|-------------|-----|-----------|-----------------|
| cp01.faekcorp.lab | control-plane | 10.70.21.6 | Ubuntu 24.04.4 LTS | 2.2.6 | v1.35.8 |
| cp02.faekcorp.lab | control-plane | 10.70.31.209 | Ubuntu 24.04.4 LTS | 2.2.6 | v1.35.8 |
| cp03.faekcorp.lab | control-plane | 10.70.41.250 | Ubuntu 24.04.4 LTS | 2.2.6 | v1.35.8 |
| worker1.faekcorp.lab | worker | 10.70.61.85 | Ubuntu 24.04.4 LTS | 2.2.6 | v1.35.8 |
| worker2.faekcorp.lab | worker | 10.70.11.67 | Ubuntu 24.04.4 LTS | 2.2.6 | v1.35.8 |

### Cluster-level configuration

| Setting | Value |
|---------|-------|
| API HA endpoint | `haproxy.faekcorp.lab:6443` |
| HAProxy IP | 10.70.11.192 |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |
| DNS domain | cluster.local |
| CNI | Cilium 1.20.1 |
| IPAM | cluster-pool |
| Routing | tunnel / Geneve |
| Service replacement | eBPF (KubeProxyReplacement=True) |
| kube-proxy | Removed (no DaemonSet, no ConfigMap) |
| etcd | 3.6.6 (stacked, 3 members) |
| Administration host | Bastion (kubectl + helm only) |

### Architecture diagram

```
                         Bastion
                    kubectl + helm
                          │
                          │ TCP 6443
                          ▼
                    ┌──────────────┐
                    │  HAProxy     │
                    │haproxy.faek  │
                    │  corp.lab    │
                    │  :6443       │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           ┌─────┐     ┌─────┐     ┌─────┐
           │cp01 │     │cp02 │     │cp03 │
           │etcd │◄───►│etcd │◄───►│etcd │
           │apis │     │apis │     │apis │
           │cm   │     │cm   │     │cm   │
           │sch  │     │sch  │     │sch  │
           └─────┘     └─────┘     └─────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    Cilium 1.20.1
                    eBPF + Geneve
                           │
              ┌────────────┼────────────┐
              ▼                         ▼
           worker1                   worker2
          10.70.61.85               10.70.11.67
```

### Important architectural note

**kubectl is NOT installed on cluster nodes.** Only on the bastion. This differs from many tutorials, which assume kubectl exists everywhere. All `kubectl` commands in this document are run from the bastion.

---

## 4. Target version selection

### Kubernetes release state at time of upgrade (2026-10-11)

| Minor | Latest patch | EOL | Notes |
|-------|--------------|-----|-------|
| 1.37 | 1.37.1 | 2027-10-28 | Newest minor |
| 1.36 | 1.36.5 | 2027-06-28 | Our target minor |
| 1.35 | 1.35.9 | 2027-02-28 | Our current minor |
| 1.34 | 1.34.12 | 2026-10-27 | Already EOL |

### Our choice: 1.36.4

**Rationale:**

- **Follow the Prod N-1 patch policy.** Within a minor line, production deployments should lag the absolute latest patch by one to absorb soak time.
- 1.36.5 was released 2026-09-15 — about 4 weeks before our upgrade.
- 1.36.4 was released 2026-08-11 — about 2 months before. Well-soaked.
- Moving *into* a new minor, we wanted the safest patch in that line, not the newest.

**Chosen version string:** `1.36.4-1.1` (the `-1.1` is the Debian package revision suffix).

### Why not 1.35.9 first?

We were on 1.35.8. 1.35.9 existed. Standard practice would be to patch to 1.35.9 then jump to 1.36.4. We skipped it because patching within the old minor buys nothing when you're about to leave it. `kubeadm upgrade` supports jumping directly.

---

## 5. Compatibility verification (containerd, Cilium, etcd)

### containerd (CRI) compatibility

| K8s version | containerd minimum |
|-------------|--------------------|
| 1.35 | 2.2.0+ |
| **1.36** | **2.3.0+, 2.2.0+** |
| 1.37 | 2.4.0+, 2.3.0+ |

**Our containerd: 2.2.6.** Satisfies `2.2.0+`. **No upgrade required.**

### Cilium (CNI) compatibility

| Cilium version | Supported Kubernetes |
|----------------|----------------------|
| 1.20 | >=1.33 <1.37 |
| 1.19 | >=1.32 <1.36 |
| 1.18 | >=1.30 <1.34 |

**Our Cilium: 1.20.1.** Supports 1.33 through 1.36. Target 1.36 is within range. **No upgrade required.**

### etcd

`kubeadm upgrade plan` reports the target etcd version. Our case: **3.6.6 → 3.6.8**. Same minor (3.6 → 3.6), patch-level change. Safe.

---

## 6. What we chose NOT to upgrade

### containerd — stayed at 2.2.6

Reasoning:
- Already in the supported range for 1.36 (`2.2.0+`)
- Upgrading containerd means restarting the CRI on all 5 nodes — a moving part we didn't need
- Cilium's eBPF datapath depends on containerd's CRI; a containerd upgrade during the K8s upgrade window adds risk
- **One change at a time**

**Documented for later:** Appendix B describes how to upgrade containerd when it's actually needed.

### Cilium — stayed at 1.20.1

Reasoning:
- Already supports 1.36
- Would have required a Helm release upgrade on the bastion plus rolling DaemonSet updates on all 5 nodes
- Adding a second moving part during the K8s upgrade window multiplies the failure surface

**Documented for later:** Appendix C describes how to verify CNI compatibility before an upgrade.

---

## 7. Version skew rules applied

Kubernetes officially allows these skews during a rolling upgrade:

| Component | Allowed skew |
|-----------|--------------|
| kube-apiserver (HA) | All within 1 minor |
| kube-controller-manager / kube-scheduler | ≤ apiserver, prefer same minor |
| kubelet | ≤ apiserver, up to 2 minors behind |
| kubectl | ±1 minor from apiserver |

**Our upgrade order respected these:**

1. All 3 control planes upgraded first — apiserver, controller-manager, scheduler move together via static pods.
2. Workers upgraded last — kubelet can lag up to 2 minors; we lagged by 1.
3. Bastion kubectl upgraded at the end — client/server match.

**Mixed-version periods during rollout were intentional and safe:**

- During cp01 upgrade: cp01 at 1.36.4, cp02/cp03 at 1.35.8
- During worker upgrades: CPs at 1.36.4, workers at 1.35.8

---

## 8. Phase 0 — pre-flight verification

Before any modification, we captured the cluster state. This serves as:
- A baseline for post-upgrade comparison
- A reference if we need to roll back
- A record of pre-existing conditions

**Commands run from bastion:**

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl version
kubectl get deploy -A
kubectl get ds -A
helm list -n kube-system
```

**Commands run from cp01:**

```bash
containerd --version
kubelet --version
kubeadm version
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table
```

---

## 9. Phase 0 — node state capture

### Baseline node state

```
NAME                   STATUS   ROLES           AGE     VERSION
cp01.faekcorp.lab      Ready    control-plane   7d18h   v1.35.8
cp02.faekcorp.lab      Ready    control-plane   7d16h   v1.35.8
cp03.faekcorp.lab      Ready    control-plane   7d15h   v1.35.8
worker1.faekcorp.lab   Ready    worker          6d18h   v1.35.8
worker2.faekcorp.lab   Ready    worker          6d18h   v1.35.8
```

**Details:**
- All 5 nodes `Ready`
- All on `v1.35.8`
- All on `Ubuntu 24.04.4 LTS`
- All running `containerd://2.2.6`

**Pod state baseline:**

```
NAMESPACE       NAME                                        READY   STATUS
after-restore   after-restore-deploy-...-q                                1/1     Running
after-restore   after-restore-deploy-...-xf4xl                            1/1     Running
default         nginx-deploy-...-brsgn                                    1/1     Running
default         nginx-deploy-...-tjwgb                                    1/1     Running
default         test-busy-...-5fj6m                                       1/1     Running
kube-system     cilium-... (5 pods)                                       1/1     Running
kube-system     cilium-envoy-... (5 pods)                                 1/1     Running
kube-system     cilium-operator-...-4vdpc                                 1/1     Running
kube-system     coredns-...-62m2j                                         1/1     Running
kube-system     coredns-...-w86rp                                         1/1     Running
kube-system     etcd-cp01/02/03.faekcorp.lab                              1/1     Running
kube-system     kube-apiserver-cp01/02/03.faekcorp.lab                    1/1     Running
kube-system     kube-controller-manager-cp01/02/03.faekcorp.lab           1/1     Running
kube-system     kube-scheduler-cp01/02/03.faekcorp.lab                    1/1     Running
```

**No kube-proxy** in the list. Good — confirms the cluster was already kube-proxy-free.

### Baseline version check

```
Client Version: v1.35.8
Kustomize Version: v5.7.1
Server Version: v1.35.8
```

Client and server matched.

---

## 10. Phase 0 — etcd cluster verification

**Command (from cp01):**

```bash
sudo etcdctl \
  --endpoints=https://10.70.21.6:2379,https://10.70.31.209:2379,https://10.70.41.250:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

**Output:**

| Endpoint | Version | Leader | Term | Raft Index | Applied Index |
|----------|---------|--------|------|-----------|---------------|
| 10.70.21.6 (cp01) | 3.6.6 | false | 3 | 85958 | 85958 |
| 10.70.31.209 (cp02) | 3.6.6 | **true** | 3 | 85958 | 85958 |
| 10.70.41.250 (cp03) | 3.6.6 | false | 3 | 85958 | 85958 |

**Interpretation:**
- All three members `started`
- All on same term and index — perfectly in sync
- cp02 was the leader
- DB size 12 MB with 4.5 MB in use (64% free)
- Quota 2.1 GB — massive headroom

**Member list also verified:**

```bash
sudo etcdctl ... member list -w table
```

All three members `started`, no learners, no unhealthy members.

**Quorum safety analysis:** 3 members, quorum = 2. We can lose exactly one member at a time. This defines the upgrade constraint: **never upgrade two control planes simultaneously.**

---

## 11. Phase 0 — kubeadm-config inspection

**Command (from bastion):**

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml
```

**Contents (before upgrade):**

```yaml
data:
  ClusterConfiguration: |-
    apiServer: {}
    apiVersion: kubeadm.k8s.io/v1beta4
    caCertificateValidityPeriod: 87600h0m0s
    certificateValidityPeriod: 8760h0m0s
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controlPlaneEndpoint: haproxy.faekcorp.lab:6443
    controllerManager: {}
    dns: {}
    encryptionAlgorithm: RSA-2048
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: registry.k8s.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.35.8
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/12
    proxy: {}
    scheduler: {}
```

**Key findings:**

1. **`controlPlaneEndpoint: haproxy.faekcorp.lab:6443`** — HA endpoint correctly configured
2. **`kubernetesVersion: v1.35.8`** — matches reality
3. **`apiVersion: kubeadm.k8s.io/v1beta4`** — modern config format
4. **`proxy: {}`** — empty. **This does NOT mean "skip kube-proxy."** It means "use default kube-proxy config."
5. **No `skipPhases`** anywhere

**CRITICAL:** finding #4 caused the main incident in this upgrade. See Part C, Section 14.

---

## 12. Phase 0 — capacity and PDB review

### Node capacity

**Command:**

```bash
kubectl describe nodes | grep -A5 -i allocated
```

| Node | CPU Requests | CPU Limits | Mem Requests | Mem Limits |
|------|-------------|-----------|--------------|-----------|
| cp01 | 750m (37%) | 1 (50%) | 110Mi (2%) | 1Gi (27%) |
| cp02 | 750m (37%) | 1 (50%) | 110Mi (2%) | 1Gi (27%) |
| cp03 | 950m (47%) | 1 (50%) | 250Mi (6%) | 1364Mi (36%) |
| worker1 | 100m (5%) | 1 (50%) | 10Mi (0%) | 1Gi (27%) |
| worker2 | 100m (5%) | 1 (50%) | 10Mi (0%) | 1Gi (27%) |

**Workers are nearly empty.** Plenty of room to absorb evicted pods during drains. CPs are `NoSchedule`-tainted so workloads won't land there during drains.

### PodDisruptionBudgets

```bash
kubectl get pdb -A
```

**Result:** `No resources found`

**Impact:**
- Drains will be **fast** — no PDBs to slow evictions
- But also: **no protection** for workloads during drains

For this lab, acceptable. In production, each critical workload would have a PDB.

### Control plane taints verified

```bash
kubectl describe node cp01.faekcorp.lab | grep -A3 Taints
```

**Result:**
```
Taints:  node-role.kubernetes.io/control-plane:NoSchedule
```

All three CPs have the standard taint. This means:
- Workloads won't land on CPs during normal scheduling
- But CoreDNS **does** tolerate this taint, so CoreDNS pods can run on CPs
- The taint affects our drain strategy (see Part C)

---

## 13. What Phase 0 told us

| Check | Status |
|-------|:---:|
| etcd cluster health | ✅ |
| etcd quorum safety (3 members, quorum=2) | ✅ |
| kubeadm-config present, correct | ✅ |
| Node capacity headroom | ✅ |
| cgroup v2 | ✅ |
| No PDBs | ⚠️ (expected for this lab) |
| `skipPhases` for kube-proxy | ❌ Not present — **must remember to skip manually** |
| CP taints present | ✅ |

**The one thing we flagged but under-weighted:** the absence of `skipPhases` for kube-proxy in the ConfigMap. We planned to use the CLI flag on cp01. **This turned out to be insufficient.** See Part C, Section 14.

---

**End of Part A.**

*Part B covers the upgrade execution: Phases 1–4, including all commands and outputs.*
*Part C covers the incidents, the controlled reboots, the final state, and all appendices.*

---

# PART B — THE UPGRADE EXECUTION (PHASES 1–4)

---

## Table of Contents — Part B

14. The `apply` vs `node` distinction
15. Phase 1 — cp01 (first control plane)
16. Phase 2a — cp02 (second control plane)
17. Phase 2b — cp03 (third control plane, and the kube-proxy moment)
18. Phase 3a — worker1
19. Phase 3b — worker2
20. Phase 4 — post-upgrade validation

---

## 14. The `apply` vs `node` distinction

This is the single most important concept in a kubeadm HA upgrade.

### `kubeadm upgrade apply`

Run on **exactly one** control-plane node (the "authority"):

- **Writes cluster-wide state**: `kubeadm-config` ConfigMap, `kubelet-config` ConfigMap, addon deployments (CoreDNS, kube-proxy), bootstrap-token RBAC
- **Then** upgrades the local node's static pod manifests
- **Only one CP should ever run `apply`**

### `kubeadm upgrade node`

Run on **every other** control-plane node, and on every worker:

- **Reads** the target version from `kubeadm-config`
- **Only** upgrades local static pod manifests (or, on workers, just kubelet config)
- **Does NOT touch** addons, RBAC, or cluster-wide ConfigMaps
- **Does NOT accept `--skip-phases`** — this is critical

### Why this design

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

### The deferred addon phase — the trap

This is where the incident began. In HA clusters, kubeadm **defers** the addon phase (CoreDNS, kube-proxy) until **all control planes are on the new version**. The output of cp01's `apply` shows:

```
[upgrade/addon] Skipping upgrade of addons because control plane instances [cp02, cp03] have not been upgraded
```

The deferred addon phase runs once, on the **last** CP's `upgrade node`. But `upgrade node`:
- Doesn't accept `--skip-phases`
- Has no memory of the flag passed to cp01's `apply`

**Result:** the deferred addon phase reinstalls kube-proxy on the last CP — with no way to prevent it via CLI. The ConfigMap must be the authority.

---

## 15. Phase 1 — cp01 (first control plane)

### Step 1 — SSH to cp01

```bash
ssh ubuntu@cp01.faekcorp.lab
```

### Step 2 — Baseline capture

```bash
kubeadm version -o short
kubelet --version
cat /etc/apt/sources.list.d/kubernetes.list
apt-mark showhold
```

**Output:**

```
v1.35.8
Kubernetes v1.35.8
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
containerd.io
kubeadm
kubelet
```

**Note:** no `kubectl` hold — kubectl isn't installed on cp01.

### Step 3 — Update APT repo

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list
```

**Output:**
```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /
```

**Note the `|` sed delimiter** instead of `/` — the repo URL contains `/`, and using `/` would require escaping each one.

### Step 4 — Refresh package index

```bash
sudo apt-get update
```

**Output (relevant line):**
```
Get:6 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.36/deb InRelease [1227 B]
```

### Step 5 — Confirm target version exists

```bash
apt-cache madison kubeadm | grep 1.36.4
```

**Output:**
```
kubeadm | 1.36.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.36/deb  Packages
```

Also checked the full range:
```bash
apt-cache madison kubeadm | grep 1.36
```

**Output:**
```
kubeadm | 1.36.5-1.1 | .../v1.36/deb  Packages
kubeadm | 1.36.4-1.1 | .../v1.36/deb  Packages
kubeadm | 1.36.3-1.1 | .../v1.36/deb  Packages
kubeadm | 1.36.2-2.1 | .../v1.36/deb  Packages
kubeadm | 1.36.1-1.1 | .../v1.36/deb  Packages
kubeadm | 1.36.0-1.1 | .../v1.36/deb  Packages
```

Confirms both target and alternatives exist.

### Step 6–8 — Install kubeadm 1.36.4

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
```

**Output (relevant):**
```
Unpacking kubeadm (1.36.4-1.1) over (1.35.8-1.1) ...
Setting up kubeadm (1.36.4-1.1) ...
kubeadm set on hold.
```

**Note:** kubelet is still 1.35.8. We only changed the *decider*.

### Step 9 — Verify

```bash
kubeadm version -o short
```

**Output:**
```
v1.36.4
```

Also with full version string:
```
kubeadm version: &version.Info{Major:"1", Minor:"36", GitVersion:"v1.36.4",
  GitCommit:"bb826b1d48562f110659e64e8ec444327433db95",
  BuildDate:"2026-08-20T03:08:41Z", GoVersion:"go1.26.5", Platform:"linux/amd64"}
```

### Step 10 — Pre-pull container images

```bash
sudo kubeadm config images pull --kubernetes-version v1.36.4
```

**Output:**
```
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.36.4
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.36.4
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.36.4
[config/images] Pulled registry.k8s.io/kube-proxy:v1.36.4
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.14.2
[config/images] Pulled registry.k8s.io/pause:3.10.2
[config/images] Pulled registry.k8s.io/etcd:3.6.8-0
```

**Verify images are now on disk:**

```bash
sudo crictl images
```

Now shows both v1.35.8 and v1.36.4 versions for each image, plus the new CoreDNS and etcd.

### Step 11 — Dry-run: `kubeadm upgrade plan`

```bash
sudo kubeadm upgrade plan v1.36.4
```

**Full output:**

```
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade/config] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W1011 13:20:14.278004    4161 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: 1.35.8
[upgrade/versions] kubeadm version: v1.36.4
[upgrade/versions] Target version: v1.36.4
[upgrade/versions] Latest version in the v1.35 series: v1.36.4
W1011 13:20:19.316694    4161 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE                   CURRENT   TARGET
kubelet     cp01.faekcorp.lab      v1.35.8   v1.36.4
kubelet     cp02.faekcorp.lab      v1.35.8   v1.36.4
kubelet     cp03.faekcorp.lab      v1.35.8   v1.36.4
kubelet     worker1.faekcorp.lab   v1.35.8   v1.36.4
kubelet     worker2.faekcorp.lab   v1.35.8   v1.36.4

Upgrade to the latest version in the v1.35 series:
COMPONENT                 NODE                CURRENT   TARGET
kube-apiserver            cp01.faekcorp.lab   v1.35.8   v1.36.4
kube-apiserver            cp02.faekcorp.lab   v1.35.8   v1.36.4
kube-apiserver            cp03.faekcorp.lab   v1.35.8   v1.36.4
kube-controller-manager   cp01.faekcorp.lab   v1.35.8   v1.36.4
kube-controller-manager   cp02.faekcorp.lab   v1.35.8   v1.36.4
kube-controller-manager   cp03.faekcorp.lab   v1.35.8   v1.36.4
kube-scheduler            cp01.faekcorp.lab   v1.35.8   v1.36.4
kube-scheduler            cp02.faekcorp.lab   v1.35.8   v1.36.4
kube-scheduler            cp03.faekcorp.lab   v1.35.8   v1.36.4
kube-proxy                                    1.35.8    v1.36.4
CoreDNS                                       v1.13.1   v1.14.2
etcd                      cp01.faekcorp.lab   3.6.6-0   3.6.8-0
etcd                      cp02.faekcorp.lab   3.6.6-0   3.6.8-0
etcd                      cp03.faekcorp.lab   3.6.6-0   3.6.8-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.36.4
_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   -                 v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

### Step 12 — Review the plan

**What we checked:**

1. **Two warnings about kube-proxy ConfigMap** — expected, since we deleted it
2. **Cluster version 1.35.8, target 1.36.4** — correct
3. **Manual kubelet upgrade required on all 5 nodes** — expected
4. **All three control-plane components listed** — apiserver, CM, scheduler
5. **`kube-proxy` in the list** — this is what we must skip
6. **CoreDNS 1.13.1 → 1.14.2** — addon upgrade
7. **etcd 3.6.6 → 3.6.8** — same minor, patch upgrade
8. **MANUAL UPGRADE REQUIRED: no for both rows** — no config migration needed
9. **Recommended command has no `--skip-phases`** — we add it manually

**Verdict:** plan clean. Proceed.

### Step 13 — Apply

```bash
sudo kubeadm upgrade apply v1.36.4 --skip-phases=addon/kube-proxy
```

**Full output:**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W1011 15:29:39.573987    5179 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found
[upgrade/preflight] Running preflight checks
[upgrade] Running cluster health checks
[upgrade/preflight] You have chosen to upgrade the cluster version to "v1.36.4"
[upgrade/versions] Cluster version: v1.35.8
[upgrade/versions] kubeadm version: v1.36.4
[upgrade] Are you sure you want to proceed? [y/N]: y

[upgrade/preflight] Pulling images required for setting up a Kubernetes cluster
[upgrade/preflight] This might take a minute or two, depending on the speed of your internet connection
[upgrade/preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[upgrade/control-plane] Upgrading your static Pod-hosted control plane to version "v1.36.4" (timeout: 5m0s)...

[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests749838350"
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-10-11-15-29-55/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available

[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-10-11-15-29-55/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!

[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-10-11-15-29-55/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!

[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2026-10-11-15-29-55/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 3 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!

[upgrade/control-plane] The control plane instance for this node was successfully upgraded!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
W1011 15:32:30.726768    5179 postupgrade.go:105] Using temporary directory /etc/kubernetes/tmp/kubeadm-kubelet-config-2026-10-11-15-32-30 for kubelet config. To override it set the environment variable KUBEADM_UPGRADE_DRYRUN_DIR
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config-2026-10-11-15-32-30/config.yaml
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!

[upgrade/bootstrap-token] Configuring bootstrap token and cluster-info RBAC rules
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Configured RBAC rules to allow the API server kubelet client certificate to access the kubelet API

[upgrade/addon] Skipping upgrade of addons because control plane instances [cp02.faekcorp.lab cp03.faekcorp.lab] have not been upgraded

[upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.36.4".
[upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
```

**What this output tells us:**

1. **etcd upgraded first** — new manifest written, certs renewed, pod restarted, "upgraded successfully"
2. **apiserver next** — certs renewed (server, kubelet-client, front-proxy, etcd-client)
3. **controller-manager and scheduler after** — in order
4. **Each component waited for health before moving on** — this is why the whole sequence is safe
5. **Cluster-wide state updated**:
   - `kubeadm-config` ConfigMap updated with new version
   - `kubelet-config` ConfigMap rewritten
   - RBAC rules refreshed
6. **`[addons]` line NOT present** — our `--skip-phases=addon/kube-proxy` worked for this run
7. **`[upgrade/addon] Skipping upgrade of addons because cp02/cp03 not upgraded`** — critical line. kubeadm is deferring addon application to the last CP's `upgrade node`.
8. **SUCCESS** — cp01's control plane is now on 1.36.4

### Bastion watch during apply

From a second terminal on bastion:

```bash
kubectl get pod -n kube-system -w
```

**Captured transitions:**

```
kube-scheduler-cp01.faekcorp.lab       1/1  Running  11 (4h13m ago)
kube-controller-manager-cp01           1/1  Running  10 (4h13m ago)
kube-apiserver-cp01                    1/1  Running  10 (4h13m ago)
etcd-cp01                              1/1  Running   9 (4h13m ago)
etcd-cp01                              1/1  Terminating
kube-apiserver-cp01                    0/1  Running
etcd-cp01                              0/1  Pending
kube-controller-manager-cp01           1/1  Running  11 (67s ago)
kube-scheduler-cp01                    0/1  Running  12 (63s ago)
kube-apiserver-cp01                    1/1  Running
etcd-cp01                              1/1  Running  0  0s
kube-scheduler-cp01                    1/1  Running  12 (64s ago)
```

**Key observations:**

- Each static pod restarted in sequence
- The watch survived the transitions — HAProxy routed the watch request to cp02/cp03 during cp01's apiserver restart
- Everything came back green

### Step 14–20 — kubelet and kubectl

**Step 15 — Unhold kubelet:**

```bash
sudo apt-mark unhold kubelet
```

**Step 16 — Install kubelet 1.36.4-1.1:**

```bash
sudo apt-get install -y kubelet=1.36.4-1.1
```

**Output:**

```
Unpacking kubelet (1.36.4-1.1) over (1.35.8-1.1) ...
Setting up kubelet (1.36.4-1.1) ...
...
Restarting services...
 systemctl restart kubelet.service
No containers need to be restarted.
```

**Note:** the .deb post-install script **automatically restarts kubelet**.

**Step 17 — Re-hold:**

```bash
sudo apt-mark hold kubelet
```

**Step 18 — Reload systemd:**

```bash
sudo systemctl daemon-reload
```

**Step 19 — Restart kubelet:**

```bash
sudo systemctl restart kubelet
```

**Step 20 — Verify:**

```bash
kubelet --version
sudo systemctl is-active kubelet.service
```

**Output:**

```
Kubernetes v1.36.4
active
```

### The kubelet restart effect

The `systemctl restart kubelet` caused **all static pods to restart again**. The `kubectl get pods -w` output showed:

```
etcd-cp01.faekcorp.lab                      0/1  ContainerCreating   14m
kube-apiserver-cp01.faekcorp.lab            0/1  ContainerCreating   13m
kube-controller-manager-cp01.faekcorp.lab   0/1  ContainerCreating   13m
kube-scheduler-cp01.faekcorp.lab            0/1  ContainerCreating   12m

etcd-cp01.faekcorp.lab                      0/1  Terminating
etcd-cp01.faekcorp.lab                      0/1  Pending
etcd-cp01.faekcorp.lab                      0/1  Running             0    0s
kube-apiserver-cp01.faekcorp.lab            0/1  Running             0    0s
kube-controller-manager-cp01.faekcorp.lab   0/1  Running             0    0s
kube-scheduler-cp01.faekcorp.lab            0/1  Running             0    0s
```

**AGE reset to ~3 minutes. These are new pods, not re-adopted.**

This is a known behavior with containerd 2.x — kubelet restart causes static pod containers to be re-created even if manifests haven't changed. Not a problem; the pods come back healthy. But worth noting.

### Verification

**From bastion:**

```bash
kubectl get nodes
```

**Output:**

```
NAME                   STATUS   ROLES           AGE     VERSION
cp01.faekcorp.lab      Ready    control-plane   7d23h   v1.36.4   ← UPGRADED
cp02.faekcorp.lab      Ready    control-plane   7d20h   v1.35.8
cp03.faekcorp.lab      Ready    control-plane   7d20h   v1.35.8
worker1.faekcorp.lab   Ready    worker          6d22h   v1.35.8
worker2.faekcorp.lab   Ready    worker          6d22h   v1.35.8
```

**cp01 is on v1.36.4. Phase 1 complete.**

---

## 16. Phase 2a — cp02 (second control plane)

### Repo change and kubeadm install

**From cp02:**

```bash
ssh ubuntu@cp02.faekcorp.lab
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list
```

**Output:**
```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /
```

**apt update:**

```bash
sudo apt-get update
```

**Install kubeadm:**

```bash
sudo apt-mark unhold kubeadm
```

**Error (first attempt, missing sudo):**
```
dpkg: error: required read/write access to the dpkg database directory /var/lib/dpkg
E: Sub-process dpkg --set-selections returned an error code (2)
E: Executing dpkg failed. Are you root?
```

**Corrected:**
```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short
```

**Output:**
```
v1.36.4
```

### The upgrade

```bash
sudo kubeadm upgrade node
```

**Full output:**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W1011 16:09:50.038906   12015 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found
[upgrade/preflight] Running pre-flight checks
[upgrade/preflight] Pulling images required for setting up a Kubernetes cluster
[upgrade/preflight] This might take a minute or two, depending on the speed of your internet connection
[upgrade/preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[upgrade/control-plane] Upgrading your Static Pod-hosted control plane instance to version "v1.36.4"...

[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests716970391"
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/etcd.yaml"
...
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available

[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
...
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!

[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
...
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!

[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
...
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!

[upgrade/control-plane] The control plane instance for this node was successfully upgraded!
[upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
[upgrade/addon] Skipping upgrade of addons because control plane instances [cp03.faekcorp.lab] have not been upgraded
[upgrade/addon] Skipping upgrade of addons because control plane instances [cp03.faekcorp.lab] have not been upgraded
```

**Key lines:**
- No `[upgrade/addon] Applied essential addon` — addons deferred
- **Two identical skip lines** — cp03 is the last CP not upgraded
- Everything else upgraded successfully

### kubelet install

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubelet --version
```

**Output:**
```
Kubernetes v1.36.4
```

### etcd verification after cp02

```bash
sudo etcdctl \
  --endpoints=https://10.70.21.6:2379,https://10.70.31.209:2379,https://10.70.41.250:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

**Output:**

| Endpoint | Version | Leader | Term | Raft Index | Applied Index |
|----------|---------|--------|------|-----------|---------------|
| 10.70.21.6 | **3.6.8** | false | 4 | 136704 | 136704 |
| 10.70.31.209 | **3.6.8** | false | 4 | 136704 | 136704 |
| 10.70.41.250 | 3.6.6 | **true** | 4 | 136704 | 136704 |

**Reading:**
- cp01 and cp02 on 3.6.8
- cp03 still on 3.6.6 (not yet upgraded)
- **Leadership moved to cp03** when cp02's etcd restarted
- **All three in sync** (same raft index/applied index)
- **Term went 3 → 4** — one leader election occurred

### Verification from bastion

```bash
kubectl get nodes
```

**Output:**

```
cp01.faekcorp.lab      Ready    control-plane   7d23h   v1.36.4
cp02.faekcorp.lab      Ready    control-plane   7d21h   v1.36.4   ← UPGRADED
cp03.faekcorp.lab      Ready    control-plane   7d20h   v1.35.8
worker1.faekcorp.lab   Ready    worker          6d23h   v1.35.8
worker2.faekcorp.lab   Ready    worker          6d23h   v1.35.8
```

**2 of 3 CPs on 1.36.4. Quorum preserved.**

---

## 17. Phase 2b — cp03 (third control plane, and the kube-proxy moment)

### Repo change and kubeadm install

**From cp03:**

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short
```

**Output:**
```
v1.36.4
```

### The upgrade — the moment of truth

```bash
sudo kubeadm upgrade node
```

**Full output:**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
W1011 16:48:12.963766    6071 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" not found
[upgrade/preflight] Running pre-flight checks
[upgrade/preflight] Pulling images required for setting up a Kubernetes cluster
[upgrade/control-plane] Upgrading your Static Pod-hosted control plane instance to version "v1.36.4"...

[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available

[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!

[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!

[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!

[upgrade/control-plane] The control plane instance for this node was successfully upgraded!
[upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!

[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy              ← THE PROBLEM
```

**Two lines at the end. Both are new:**

1. **`[addons] Applied essential addon: CoreDNS`** — expected. CoreDNS upgrades when the last CP completes.
2. **`[addons] Applied essential addon: kube-proxy`** — unexpected. **kube-proxy reinstalled.**

**This is where the incident begins.** See Part C, Section 14 for the full analysis.

### kubelet install

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Immediate state check

**From bastion:**

```bash
kubectl get nodes
```

**Output:**

```
cp01.faekcorp.lab      Ready    control-plane   8d      v1.36.4
cp02.faekcorp.lab      Ready    control-plane   7d21h   v1.36.4
cp03.faekcorp.lab      Ready    control-plane   7d21h   v1.36.4   ← UPGRADED
worker1.faekcorp.lab   Ready    worker          6d23h   v1.35.8
worker2.faekcorp.lab   Ready    worker          6d23h   v1.35.8
```

**All 3 CPs on 1.36.4.**

**But check pods:**

```bash
kubectl -n kube-system get pods | grep -E 'coredns|kube-proxy'
```

**Output:**

```
coredns-589f44dc88-5bbgz                    0/1     Running   0               3m51s
coredns-589f44dc88-nvn2r                    0/1     Running   0               3m51s
coredns-7d764666f9-w86rp                    1/1     Running   7 (5h39m ago)   8d
kube-proxy-8kwqw                            1/1     Running   0               4m14s
kube-proxy-cvtvn                            1/1     Running   0               4m14s
kube-proxy-cw54f                            1/1     Running   0               4m14s
kube-proxy-kv2xl                            1/1     Running   0               4m14s
kube-proxy-npztq                            1/1     Running   0               4m14s
```

**Two problems visible:**
1. **5 kube-proxy pods running** — one per node
2. **2 new CoreDNS pods stuck at `0/1`** — not Ready

The old CoreDNS pod (1.13.1) is still running — it's keeping DNS alive.

**This is where Phase 3 begins with a detour.** We need to fix these before upgrading workers. See Part C.

**End of Part B.**

*Part C covers: the kube-proxy incident in detail, the ConfigMap fix, Phase 3 (workers), Phase 4 (validation), the controlled reboots, the cp01 kernel incident, final state, and all appendices.*

---

# PART C — INCIDENTS, WORKERS, VALIDATION, REBOOTS, AND APPENDICES

---

## Table of Contents — Part C

**18. The kube-proxy Incident**
- 18.1 Timeline
- 18.2 Immediate symptoms
- 18.3 CoreDNS fallout (full 503 logs)
- 18.4 iptables discovery
- 18.5 iptables cleanup
- 18.6 CoreDNS restart
- 18.7 kube-proxy removal
- 18.8 ConfigMap fix attempt #1 (skipPhases — wrong)
- 18.9 The unmarshal error
- 18.10 ConfigMap fix attempt #2 (proxy.disabled — right)
- 18.11 Verification

**19. Phase 3a — worker1**
**20. Phase 3b — worker2**
**21. Phase 4 — post-upgrade validation**
**22. Bastion kubectl upgrade**
**23. The controlled reboot**
**24. The cp01 kernel incident**
**25. Final state**
**26. Appendices**

---

## 18. The kube-proxy Incident

This is the central incident of the upgrade. It has two root causes that combined to produce a specific failure.

### 18.1 Timeline

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

### 18.2 Immediate symptoms

**Symptom 1 — kube-proxy pods running:**

```bash
kubectl -n kube-system get pods | grep kube-proxy
```

```
kube-proxy-8kwqw   1/1  Running  0  4m14s
kube-proxy-cvtvn   1/1  Running  0  4m14s
kube-proxy-cw54f   1/1  Running  0  4m14s
kube-proxy-kv2xl   1/1  Running  0  4m14s
kube-proxy-npztq   1/1  Running  0  4m14s
```

**Symptom 2 — new CoreDNS pods not Ready:**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```

```
coredns-589f44dc88-5bbgz   0/1  Running  0  5m1s  10.244.4.194  worker2
coredns-589f44dc88-nvn2r   0/1  Running  0  5m1s  10.244.3.70   worker1
coredns-7d764666f9-w86rp   1/1  Running  7  8d     10.244.2.94   cp03
```

**The old CoreDNS pod on cp03 (1.13.1) was still running and serving DNS.** This is why the cluster didn't fully lose DNS.

### 18.3 CoreDNS fallout — full diagnostics

**Event on the stuck pod:**

```bash
kubectl -n kube-system describe pod coredns-589f44dc88-5bbgz | tail -10
```

```
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  14m                   default-scheduler  Successfully assigned to worker2
  Normal   Pulled     14m                   kubelet            Successfully pulled image "registry.k8s.io/coredns/coredns:v1.14.2"
  Normal   Created    14m                   kubelet            Container created
  Normal   Started    14m                   kubelet            Container started
  Warning  Unhealthy  4m34s (x68 over 14m)  kubelet            Readiness probe failed:
             HTTP probe failed with statuscode: 503
```

**Pod logs:**

```bash
kubectl -n kube-system logs coredns-589f44dc88-5bbgz
```

```
maxprocs: Leaving GOMAXPROCS=2: CPU quota undefined
[INFO] plugin/kubernetes: waiting for Kubernetes API before starting server
[INFO] plugin/ready: Plugins not ready: "kubernetes"
[INFO] plugin/kubernetes: waiting for Kubernetes API before starting server
[INFO] plugin/ready: Plugins not ready: "kubernetes"
...
[WARNING] plugin/kubernetes: starting server with unsynced Kubernetes API
.:53
[INFO] plugin/reload: Running configuration SHA512 = 1b226df7...
CoreDNS-1.14.2
linux/amd64, go1.26.1, dd1df4f
[INFO] plugin/ready: Plugins not ready: "kubernetes"
[INFO] plugin/ready: Plugins not ready: "kubernetes"
...
```

**Reading the logs:**

- CoreDNS started its DNS server on port 53
- But its `kubernetes` plugin **never synced with the API**
- The `plugin/ready` endpoint kept returning "not ready" → readiness probe 503
- The warning `starting server with unsynced Kubernetes API` confirms the plugin gave up waiting

**The `kubernetes` plugin needs to reach `https://kubernetes.default.svc.cluster.local` (ClusterIP 10.96.0.1:443).** If it can't reach the API, it can't sync, and it stays not Ready.

### 18.4 iptables discovery

**Verification that the kube-proxy iptables rules were still present:**

**On worker1:**

```bash
sudo iptables-save | grep -c KUBE
```

**Output:**
```
94
```

**Inspect the relevant rules:**

```bash
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
```

**Output:**

```
Chain KUBE-SERVICES (2 references)
num  target                     prot opt source      destination
1    KUBE-SVC-NPX46M4PTMTKRN6Y  6    --   0.0.0.0/0   10.96.0.1     /* default/kubernetes:https cluster IP */ tcp dpt:443
2    KUBE-SVC-HL5LMXD5JFHQZ6LN  6    --   0.0.0.0/0   10.104.206.100 /* default/nginx-svc cluster IP */ tcp dpt:80
3    KUBE-SVL-NZTS37XDTDNXGCKJ  6    --   0.0.0.0/0   10.109.87.252  /* kube-system/hubble-peer:peer-service cluster IP */ tcp dpt:443
4    KUBE-SVC-TCOU7JCQXEZGVUNU  17   --   0.0.0.0/0   10.96.0.10    /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
5    KUBE-SVC-ERIFXISQEP7F7OF4  6    --   0.0.0.0/0   10.96.0.10    /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
6    KUBE-SVC-JD5MR3NA4I4DYORP  6    --   0.0.0.0/0   10.96.0.10    /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
7    KUBE-NODEPORTS             0    --   0.0.0.0/0   0.0.0.0/0     /* kubernetes service nodeports */ ADDRTYPE match dst-type LOCAL
```

**The smoking gun:** rule #1 — `10.96.0.1:443` was being intercepted by kube-proxy's `KUBE-SVC-NPX46M4PTMTKRN6Y` chain, **before** Cilium's eBPF could handle it.

Also:

```
:KUBE-PROXY-CANARY - [0:0]
:KUBE-PROXY-FIREWALL - [0:0]
-A OUTPUT -m conntrack --ctstate NEW -m comment --comment "kubernetes load balancer firewall" -j KUBE-PROXY-FIREWALL
```

kube-proxy's firewall and canary chains were also present.

**Conclusion:** kube-proxy's iptables rules were intercepting ClusterIP traffic. The stale DNAT rules couldn't reach the API correctly (kube-proxy state was inconsistent since the DS was just created and pods were still starting), so new CoreDNS pods couldn't sync.

### 18.5 iptables cleanup

**On every node (cp01, cp02, cp03, worker1, worker2):**

```bash
sudo iptables-save | grep -v KUBE | sudo iptables-restore
```

**What this does:**
1. Saves all current iptables rules
2. Filters out any line containing `KUBE`
3. Restores the filtered ruleset

**Result on each node:**

```bash
sudo iptables-save | grep -c KUBE
```

**Output:**
```
0
```

**Before: 94 rules per node. After: 0.**

**Note on KUBE-KUBELET-CANARY:** the kubelet sometimes regenerates this single canary chain. If a small non-zero count remains, check which chains:

```bash
sudo iptables-save | grep KUBE
```

If the only remaining entry is `KUBE-KUBELET-CANARY`, that's fine — it's kubelet's own, not kube-proxy's.

### 18.6 CoreDNS restart

**With iptables cleaned, the stuck CoreDNS pods were restarted:**

```bash
kubectl -n kube-system delete pod coredns-589f44dc88-5bbgz coredns-589f44dc88-nvn2r
```

**The Deployment recreated them:**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```

```
coredns-589f44dc88-844bl   1/1  Running  0  15s  10.244.1.249  cp02
coredns-589f44dc88-pqkvw   0/1  Running  0  15s  10.244.0.136  cp01
```

One came up Ready immediately. The other took a bit longer but eventually also reached `1/1`.

**Why the restart worked this time:** the new pod's kubernetes plugin could now reach `10.96.0.1:443` via Cilium's eBPF path (no more kube-proxy iptables interference). The plugin synced with the API. Readiness probe passed.

**Old CoreDNS pod (`coredns-7d764666f9-w86rp`) was terminated automatically by the Deployment once both new pods were Ready.**

### 18.7 kube-proxy removal

**Remove the reinstalled kube-proxy DaemonSet and ConfigMap:**

```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
```

**Output:**
```
daemonset.apps "kube-proxy" deleted from kube-system namespace
configmap "kube-proxy" deleted from kube-system namespace
```

**Verification:**

```bash
kubectl get ds -A
```

```
NAMESPACE     NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
kube-system   cilium         5         5         5       5            5
kube-system   cilium-envoy   5         5         5       5            5
```

**No kube-proxy.** Only Cilium.

```bash
kubectl get cm -n kube-system | grep kube-proxy
```

**No output.** ConfigMap gone.

### 18.8 ConfigMap fix attempt #1 — `skipPhases` (wrong)

**First we tried adding `skipPhases` to the `ClusterConfiguration`:**

```yaml
    proxy: {}
    scheduler: {}
    skipPhases:
    - addon/kube-proxy
```

**How we made the edit — the failed first attempt:**

```bash
kubectl -n kube-system get cm kubeadm-config -o json \
  | jq '.data.ClusterConfiguration += "\n  skipPhases:\n  - addon/kube-proxy"' \
  | kubectl apply -f -
```

**Result — malformed ConfigMap:**

```yaml
    proxy: {}
    scheduler: {}

      skipPhases:
      - addon/kube-proxy
```

**Two problems:**

1. **Blank line** — the original value ended with a newline, and the `+=` append added another
2. **Wrong indentation** — `skipPhases` was at 2 spaces but the existing fields were at 0; the file also had 4-space base indent for the block scalar

**We should not have used `jq` inline.** Extracting the value to a file and editing it would have been safer. We caught this and restored from backup:

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml > /tmp/kubeadm-config-backup-*.yaml
kubectl -n kube-system replace cm kubeadm-config --filename=/tmp/kubeadm-config-backup-*.yaml
```

**Then we re-edited using `kubectl edit`:**

```bash
kubectl -n kube-system edit cm kubeadm-config
```

**Resulting state:**

```yaml
    proxy: {}
    scheduler: {}
    skipPhases:
    - addon/kube-proxy
```

**This looked correct syntactically.** But it produced a new problem.

### 18.9 The unmarshal error

**When worker1 ran `kubeadm upgrade node` after the edit:**

```
W1011 19:33:42.644167   15498 cluster.go:130] error unmarshaling configuration
schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta4", Kind:"ClusterConfigura...
```

**What this means:** kubeadm tried to unmarshal the `ClusterConfiguration` and failed.

**Root cause:** `skipPhases` **is not a valid field** in `ClusterConfiguration`. It belongs in `InitConfiguration` or `JoinConfiguration`. The API server accepted the edit (it stores strings, doesn't validate embedded YAML), but kubeadm rejected it at parse time.

**Consequences:**
- The warning appeared on every `upgrade node`
- The field was effectively ignored
- The upgrade still worked because kubeadm falls back to defaults

**We then researched the correct field.** The kubeadm v1beta4 docs confirm:

> `ClusterConfiguration.dns.disabled` and `ClusterConfiguration.proxy.disabled` can be used to disable the CoreDNS and kube-proxy addons during cluster initialization.

**Also:** a known kubeadm bug (issue #3137) means `proxy.disabled: true` may be parsed by the plan phase but not honored by `apply`. So it's semantically correct but not yet reliable.

### 18.10 ConfigMap fix attempt #2 — `proxy.disabled: true` (right)

**Remove `skipPhases`, replace `proxy: {}` with the correct field.**

**Edited ConfigMap:**

```bash
kubectl -n kube-system edit cm kubeadm-config
```

**Changed:**

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

**Verified:**

```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | tail -8
```

**Output:**

```
kubernetesVersion: v1.36.4
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
proxy:
  disabled: true
scheduler: {}
```

**No `skipPhases`.** No unmarshal warning will be produced.

### 18.11 Verification on worker1

**Re-ran `kubeadm upgrade node` on worker1:**

```bash
sudo kubeadm upgrade node
```

**Output:**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W1011 20:17:18.730148   16360 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" is forbidden: User "system:node:worker1.faekcorp.lab" cannot get resource "configmaps" in API group "" in the namespace "kube-system": no relationship found between node 'worker1.faekcorp.lab' and this object
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

**The unmarshal error is gone.** The remaining warning is benign:
- Workers don't have RBAC to read ConfigMaps in kube-system (correct security posture)
- There is no kube-proxy ConfigMap anyway
- kubeadm says "continuing without it" and proceeds

**Lesson:** `proxy.disabled: true` is the correct field, and it eliminates the warning. But because of the known kubeadm bug, it may not actually prevent kube-proxy reinstalls on future upgrades. **Document this and always verify post-upgrade.**

---

## 19. Phase 3a — worker1

### Cordon + drain

**From bastion:**

```bash
kubectl cordon worker1.faekcorp.lab
kubectl drain worker1.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**Output:**

```
node/worker1.faekcorp.lab cordoned
node/worker1.faekcorp.lab already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/cilium-envoy-tlwfc, kube-system/cilium-zwfng
evicting pod default/test-busy-65dfd65b56-5fj6m
evicting pod after-restore/after-restore-deploy-759c7fb75b-xf4xl
evicting pod default/nginx-deploy-8b9dbd8c9-tjwgb
pod/after-restore-deploy-759c7fb75b-xf4xl evicted
pod/nginx-deploy-8b9dbd8c9-tjwgb evicted
pod/test-busy-65dfd65b56-5fj6m evicted
node/worker1.faekcorp.lab drained
```

**Workloads migrated to worker2 in seconds.**

### Upgrade on worker1

**From worker1:**

```bash
ssh ubuntu@worker1.faekcorp.lab
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.36.4-1.1
sudo apt-mark hold kubeadm
kubeadm version -o short       # → v1.36.4
```

**Upgrade:**

```bash
sudo kubeadm upgrade node
```

**Output:**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade/preflight] Running pre-flight checks
[upgrade/preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config.../kubelet-config-backup
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

**Note:** workers skip all control-plane and addon phases. Only the kubelet config is touched.

**kubelet install:**

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubelet --version              # → Kubernetes v1.36.4
sudo systemctl is-active kubelet.service    # → active
```

### Uncordon

```bash
kubectl uncordon worker1.faekcorp.lab
kubectl get nodes
```

**Output:**

```
cp01.faekcorp.lab      Ready    control-plane   8d     v1.36.4
cp02.faekcorp.lab      Ready    control-plane   8d     v1.36.4
cp03.faekcorp.lab      Ready    control-plane   8d     v1.36.4
worker1.faekcorp.lab   Ready    worker          7d3h   v1.36.4   ← UPGRADED
worker2.faekcorp.lab   Ready    worker          7d3h   v1.35.8
```

**4 of 5 nodes upgraded.**

---

## 20. Phase 3b — worker2

### Cordon

```bash
kubectl cordon worker2.faekcorp.lab
```

### First drain attempt — failed

```bash
kubectl drain worker2.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**Output:**

```
error: unable to drain node "worker2.faekcorp.lab" due to error:
cannot delete Pods that declare no controller (use --force to override):
default/debug-pod
There are pending nodes to be drained:
 worker2.faekcorp.lab
cannot delete Pods that declare no controller (use --force to override): default/debug-pod
```

**Cause:** an earlier debugging session left a bare Pod (`debug-pod`) on worker2. Bare Pods have no controller, so drain can't reschedule them.

**Fix:**

```bash
kubectl delete pod debug-pod
```

### Second drain attempt — succeeded

```bash
kubectl drain worker2.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**Output:**

```
node/worker2.faekcorp.lab already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/cilium-6zsqj, kube-system/cilium-envoy-rtszj
evicting pod post-upgrade-test/client
evicting pod after-restore/after-restore-deploy-759c7fb75b-7bnm8
evicting pod after-restore/after-restore-deploy-759c7fb75b-r4s7b
evicting pod default/nginx-deploy-8b9dbd8c9-g5djz
evicting pod default/nginx-deploy-8b9dbd8c9-qp5s8
evicting pod default/test-busy-65dfd65b56-m57x9
evicting pod post-upgrade-test/after-upgrade-5dd8bbbfdf-2vg5r
evicting pod post-upgrade-test/after-upgrade-5dd8bbbfdf-v6rl2
evicting pod post-upgrade-test/after-upgrade-bfdf-vqrkt
pod/... evicted
node/worker2.faekcorp.lab drained
```

**9 pods evicted.** All migrated to worker1.

### Upgrade on worker2

Same sequence as worker1.

**`kubeadm upgrade node` on worker2 (with the fixed ConfigMap):**

```
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
W1011 20:33:21.474743   16480 configset.go:77] Warning: No kubeproxy.config.k8s.io/v1alpha1 config is loaded. Continuing without it: configmaps "kube-proxy" is forbidden: User "system:node:worker2.faekcorp.lab" cannot get resource "configmaps" in API group "" in the namespace "kube-system": no relationship found between node 'worker2.faekcorp.lab' and this object
[upgrade/preflight] Running pre-flight checks
[upgrade/preflight] Skipping prepull. Not a control plane node.
[upgrade/preflight] Skipping phase. Not a control plane node.
...
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
[upgrade/addon] Skipping the addon/coredns phase. Not a control plane node.
[upgrade/addon] Skipping the addon/kube-proxy phase. Not a control plane node.
```

**No unmarshal error.** ConfigMap fix confirmed on both workers.

**kubelet install:**

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.36.4-1.1
sudo apt-mark hold kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Uncordon

```bash
kubectl uncordon worker2.faekcorp.lab
kubectl get nodes
```

**Output:**

```
cp01.faekcorp.lab      Ready    control-plane   8d     v1.36.4
cp02.faekcorp.lab      Ready    control-plane   8d     v1.36.4
cp03.faekcorp.lab      Ready    control-plane   8d     v1.36.4
worker1.faekcorp.lab   Ready    worker          7d3h   v1.36.4
worker2.faekcorp.lab   Ready    worker          7d3h   v1.36.4   ← UPGRADED
```

**All 5 nodes on v1.36.4.** ✅

---

## 21. Phase 4 — post-upgrade validation

### 21.1 Cilium health

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status
```

**Output:**

```
KVStore:                 Disabled
Kubernetes:              Ok         1.36 (v1.36.4) [linux/amd64]
Kubernetes APIs:         ["cilium/v2::CiliumCIDRGroup", "cilium/v2::CiliumClusterwideNetworkPolicy",
                          "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy",
                          "cilium/v2::CiliumNode", "core/v1::Pods", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:    True   [enp39s0   10.70.41.250 fe80::cff:c9ff:fe12:bccf (Direct Routing)]
Host firewall:           Disabled
SRv6:                    Disabled
CNI Chaining:            none
CNI Config file:         successfully wrote CNI configuration file to /host/etc/cni/net.d/05-cilium.conflist
Cilium:                  Ok   1.20.1 (v1.20.1-7d68cfb3)
NodeMonitor:             Listening for events on 2 CPUs with 64x4096 of shared memory
Cilium health daemon:    Ok
IPAM:                    IPv4: 3/254 allocated from 10.244.2.0/24
...
Routing:                 Network: Tunnel [geneve]   Host: Legacy
Attach Mode:             TCX
Device Mode:             veth
Masquerading:            IPTables [IPv4: Enabled, IPv6: Disabled]
Controller Status:       18/18 healthy
Cluster health:          5/5 reachable
Modules Health:          Stopped(0) Degraded(0) OK(76)
```

**Verification:**

- **`Kubernetes: Ok 1.36 (v1.36.4)`** — Cilium recognizes the upgraded API version
- **`KubeProxyReplacement: True`** — eBPF service path active
- **`Cilium: Ok 1.20.1`** — CNI healthy
- **`Routing: Tunnel [geneve]`** — Geneve tunneling still in use
- **`Cluster health: 5/5 reachable`** — all 5 nodes in the Cilium mesh
- **`Controller Status: 18/18 healthy`** — all internal controllers converged
- **`Modules Health: Stopped(0) Degraded(0) OK(76)`** — no degraded modules

### 21.2 etcd cluster

```bash
sudo etcdctl \
  --endpoints=https://10.70.21.6:2379,https://10.70.31.209:2379,https://10.70.41.250:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

**Output:**

| Endpoint | Version | Leader | Term | Raft Index | Applied Index |
|----------|---------|--------|------|-----------|---------------|
| 10.70.21.6 | **3.6.8** | true | 5 | 200951 | 200951 |
| 10.70.31.209 | **3.6.8** | false | 5 | 200951 | 200951 |
| 10.70.41.250 | **3.6.8** | false | 5 | 200951 | 200951 |

**Verification:**
- All three on 3.6.8 ✅
- Same term (5) — consistent cluster state
- Same raft index and applied index — no lag
- No errors
- cp01 is the leader

### 21.3 ClusterIP functional test

```bash
kubectl create ns post-upgrade-test
kubectl -n post-upgrade-test create deploy after-upgrade --image=nginx --replicas=3
kubectl -n post-upgrade-test expose deploy after-upgrade --port 80
```

**Verify pods spread across workers:**

```
after-upgrade-5dd8bbbfdf-2vg5r   1/1  Running  worker2
after-upgrade-5dd8bbbfdf-fjh5k   1/1  Running  worker1
after-upgrade-5dd8bbbfdf-vqrkt   1/1  Running  worker2
```

**Service:**

```
Name:      after-upgrade
Type:      ClusterIP
IP:        10.111.39.187
Endpoints: 10.244.4.175:80, 10.244.4.182:80, 10.244.3.124:80
```

**HTTP test through ClusterIP:**

```bash
kubectl -n post-upgrade-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://after-upgrade.post-upgrade-test.svc.cluster.local | head -5
```

**Output:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

**✅ ClusterIP works end-to-end. The request went through Cilium's eBPF service path (no kube-proxy exists).**

### 21.4 DNS test

**First test — busybox, gave a false alarm:**

```bash
kubectl run dns-test --image=busybox:latest --restart=Never --rm -it -- \
  nslookup kubernetes.default
```

**Output:**

```
Server:    10.96.0.10
Address:   10.96.0.10:53

** server can't find kubernetes.default: NXDOMAIN
** server can't find kubernetes.default: NXDOMAIN
```

**The issue:** busybox's `nslookup` does **not** expand search domains. It sends the exact name `kubernetes.default.` as a single query, which doesn't exist.

**Correct test — with netshoot (full bind-utils):**

```bash
kubectl run debug-pod --image=nicolaka/netshoot:latest -- sleep 3600
kubectl exec -it debug-pod -- sh
```

**Inside:**

```
~ # cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local faekcorp.lab ec2.internal
nameserver 10.96.0.10
options ndots:5

~ # nslookup kubernetes.default
Name:   kubernetes.default.svc.cluster.local
Address: 10.96.0.1

~ # nslookup after-upgrade.post-upgrade-test
Name:   after-upgrade.post-upgrade-test.svc.cluster.local
Address: 10.111.39.187
```

**✅ DNS works perfectly.** Search domains expand correctly. CoreDNS serves the records.

**Note:** `Got recursion not available from 10.96.0.10` warnings appear — **expected**. CoreDNS is authoritative for cluster zones, not recursive. The `Name:` and `Address:` lines are the correct answer.

### 21.5 API readyz

```bash
kubectl get --raw='/readyz?verbose'
```

**Output (abbreviated):**

```
[+]ping ok
[+]log ok
[+]etcd ok
[+]etcd-readiness ok
[+]informer-sync ok
[+]poststarthook/start-apiserver-admission-initializer ok
...
[+]shutdown ok
readyz check passed
```

**All checks pass.** API is fully functional.

---

## 22. Bastion kubectl upgrade

```bash
sudo sed -i 's|v1\.35|v1.36|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-mark unhold kubectl
sudo apt-get install -y kubectl=1.36.4-1.1
sudo apt-mark hold kubectl
kubectl version
```

**Output:**

```
Client Version: v1.36.4
Kustomize Version: v5.8.1
Server Version: v1.36.4
```

**Client and server matched.** Bastion kubectl upgraded.

---

## 23. The controlled reboot

### 23.1 Why we reboot

After a Kubernetes upgrade, a controlled reboot of each node is recommended for three reasons:

1. **Pending kernel updates** — apt may have installed a new kernel without rebooting to activate it
2. **kubelet restarts leave transient state** — static pods restarted multiple times; a clean reboot consolidates
3. **Kubelet config changes** — a reboot ensures kubelet starts with the new config from a cold baseline

### 23.2 Pre-reboot kernel status

| Node | Kernel before reboots | Needs reboot? |
|------|----------------------|:---:|
| cp01 | 7.0.0-1013-aws | ✅ |
| cp02 | 7.0.0-1013-aws | ✅ |
| cp03 | 7.0.0-1014-aws | already current |
| worker1 | 7.0.0-1013-aws | ✅ |
| worker2 | 7.0.0-1013-aws | ✅ |

**cp03 was already on the new kernel.** We rebooted it anyway for state consolidation.

### 23.3 worker1 reboot

**From bastion:**

```bash
kubectl cordon worker1.faekcorp.lab
kubectl drain worker1.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**Reboot:**

```bash
ssh ubuntu@worker1.faekcorp.lab
sudo reboot
```

**From another terminal, watch:**

```bash
kubectl get nodes -w
```

**worker1 went `NotReady,SchedulingDisabled` → `Ready,SchedulingDisabled` in ~60 seconds.**

**Verify:**

```bash
ssh ubuntu@worker1.faekcorp.lab uname -r
```

**Output:**
```
7.0.0-1014-aws
```

**Uncordon:**

```bash
kubectl uncordon worker1.faekcorp.lab
```

### 23.4 worker2 reboot

Same procedure, plus one fix: the earlier `debug-pod` had to be deleted before the drain.

```bash
kubectl cordon worker2.faekcorp.lab
kubectl delete pod debug-pod
kubectl drain worker2.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
ssh ubuntu@worker2.faekcorp.lab sudo reboot
# wait for Ready
ssh ubuntu@worker2.faekcorp.lab uname -r   # → 7.0.0-1014-aws
kubectl uncordon worker2.faekcorp.lab
```

### 23.5 cp02 reboot and the CoreDNS race

**From bastion:**

```bash
kubectl cordon cp02.faekcorp.lab
kubectl drain cp02.faekcorp.lab --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**The drain evicted a CoreDNS pod on cp02:**

```
evicting pod kube-system/coredns-589f44dc88-844bl
pod/coredns-589f44dc88-844bl evicted
```

**Reboot:**

```bash
ssh -t ubuntu@cp02.faekcorp.lab 'sudo reboot'
```

**Why `-t`:** forces TTY allocation, which matters for `sudo`. Without it, some SSH configurations close the session before `sudo reboot` fully executes.

**Verify:**

```bash
ssh ubuntu@cp02.faekcorp.lab uname -r   # → 7.0.0-1014-aws
```

**Uncordon:**

```bash
kubectl uncordon cp02.faekcorp.lab
```

**A few minutes later, a CoreDNS pod landed on worker2 and stuck at `0/1`:**

```
coredns-589f44dc88-2gxzf   0/1  Running  24m   worker2
coredns-589f44dc88-2w9fp   1/1  Running  6m8s  cp01
```

**Diagnosis:** worker2 was still stabilizing after its own reboot when the pod was scheduled there. Its kubelet and/or Cilium agent weren't fully ready. The pod's kubernetes plugin couldn't sync with the API.

**The fix:** wait. The pod's readiness probe retries continuously. Within a couple of minutes, worker2 was fully back, and the pod either recovered on its own or was replaced.

**Observed final state:**

```
coredns-589f44dc88-2w9fp   1/1  Running  cp01
coredns-589f44dc88-j9pnf   1/1  Running  worker2    ← new, Ready
```

**Both replicas Ready.** DNS healthy.

**Lesson:** during rolling reboots, pods can be scheduled onto nodes that aren't fully ready. The pod will appear `Running` but fail readiness until the node stabilizes. **Wait 2–3 minutes before intervening.**

### 23.6 cp01 reboot — the half-reboot incident

**First attempt:**

```bash
ssh ubuntu@cp01.faekcorp.lab
sudo reboot
```

**The SSH session closed. Then the bastion session came back too fast.** Subsequent `uname -r` showed the old kernel — the reboot hadn't actually taken.

**Root cause:** the `reboot` command was interrupted by the SSH session closing before it could execute. The system received the shutdown signal but hadn't fully begun the shutdown.

**Fix — use `ssh -t`:**

```bash
ssh -t ubuntu@cp01.faekcorp.lab 'sudo reboot'
```

**Or, more reliable — delayed reboot:**

```bash
ssh ubuntu@cp01.faekcorp.lab 'sudo shutdown -r +1 "Kernel update reboot"'
```

The delayed form schedules a reboot 1 minute in the future. The SSH session exits cleanly, then the reboot executes.

---

## 24. The cp01 kernel incident

After all reboots, cp01 was **still on kernel 7.0.0-1013-aws** while the other four nodes showed `7.0.0-1014-aws`.

### 24.1 Initial diagnosis

**Check installed kernels:**

```bash
dpkg -l | grep linux-image
```

**Output:**

```
ii  linux-image-6.17.0-1017-aws        6.17.0-1017.17~24.04.1  amd64  Signed kernel image aws
ii  linux-image-7.0.0-1013-aws         7.0.0-1013.13~24.04.1   amd64  Signed kernel image aws
ii  linux-image-aws                    7.0.0-1013.13~24.04.1   amd64  Linux kernel image for AWS
```

**Key finding: `linux-image-7.0.0-1014-aws` is NOT in the installed list on cp01.**

**Reboots can't fix this.** The 1014 kernel package was never installed on cp01.

### 24.2 Why this happened

The other four nodes had `linux-image-aws` pointing at the new 1014 kernel. cp01 did not. Most likely, at some earlier point:
- An `apt-get upgrade` on the other nodes pulled in the new metapackage → 1014 kernel
- On cp01, the same command either didn't run or was interrupted
- The metapackage on cp01 stayed at 1013

### 24.3 GRUB config check

```bash
sudo grub-editenv list
sudo grep -E '^GRUB_DEFAULT|^GRUB_TIMEOUT' /etc/default/grub
```

**Output:**

```
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
```

**Menu entries:**

```bash
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub/grub.cfg
```

**Output:**
```
0 : Ubuntu
```

Standard Ubuntu GRUB: one "Ubuntu" entry, which boots whatever `/boot/vmlinuz` points at. The symlink is managed by the `linux-image-aws` metapackage.

### 24.4 The fix

**Check what's available:**

```bash
apt-cache policy linux-image-aws linux-image-7.0.0-1014-aws
```

**Output:**

```
linux-image-aws:
  Installed: 7.0.0-1013.13~24.04.1
  Candidate: 7.0.0-1014.14~24.04.1        ← newer version is available

linux-image-7.0.0-1014-aws:
  Installed: (none)                        ← not installed
  Candidate: 7.0.0-1014.14~24.04.1        ← available to install
```

**Install the kernel:**

```bash
sudo apt-get install -y linux-image-aws linux-headers-aws
```

**Output:**

```
The following additional packages will be installed:
  linux-aws linux-aws-7.0-headers-7.0.0-1014 linux-aws-7.0-tools-7.0.0-1014
  linux-headers-7.0.0-1014-aws linux-image-7.0.0-1014-aws linux-modules-7.0.0-1014-aws
...
```

**Verify the symlink:**

```bash
ls -la /boot/vmlinuz /boot/vmlinuz.old
```

**Output:**

```
lrwxrwxrwx 1 root root 22 Oct 11 23:21 /boot/vmlinuz -> vmlinuz-7.0.0-1014-aws
lrwxrwxrwx 1 root root 22 Oct 11 23:21 /boot/vmlinuz.old -> vmlinuz-7.0.0-1013-aws
```

**The symlink now points at the new kernel.** GRUB will boot it on next reboot.

**Reboot:**

```bash
sudo reboot
```

**Verify:**

```bash
uname -r
```

**Output:**
```
7.0.0-1014-aws
```

**The fix worked.**

### 24.5 Final kernel state

| Node | Kernel |
|------|--------|
| cp01 | 7.0.0-1014-aws ✅ |
| cp02 | 7.0.0-1014-aws ✅ |
| cp03 | 7.0.0-1014-aws ✅ |
| worker1 | 7.0.0-1014-aws ✅ |
| worker2 | 7.0.0-1014-aws ✅ |

**All nodes identical.**

### 24.6 Lesson

When a kernel update appears "stuck", **check what's actually installed** (`dpkg -l | grep linux-image`) before assuming the reboot isn't taking. An uninstalled kernel will never boot, no matter how many reboots you do.

---

## 25. Final state

### 25.1 Nodes

```
NAME                   STATUS   ROLES           AGE    VERSION   KERNEL
cp01.faekcorp.lab      Ready    control-plane   8d     v1.36.4   7.0.0-1014-aws
cp02.faekcorp.lab      Ready    control-plane   8d     v1.36.4   7.0.0-1014-aws
cp03.faekcorp.lab      Ready    control-plane   8d     v1.36.4   7.0.0-1014-aws
worker1.faekcorp.lab   Ready    worker          7d6h   v1.36.4   7.0.0-1014-aws
worker2.faekcorp.lab   Ready    worker          7d6h   v1.36.4   7.0.0-1014-aws
```

**Every node:**
- Ready ✅
- Kubernetes v1.36.4 ✅
- Kernel 7.0.0-1014-aws ✅
- containerd 2.2.6 ✅

### 25.2 System pods

```bash
kubectl get pods -A | grep -vE 'Running|Completed'
```

**Output:** *(empty)*

**No non-Running pods.**

### 25.3 CoreDNS

```
coredns-589f44dc88-fjqkk   1/1  Running  0  19m  cp02
coredns-589f44dc88-p67fx   1/1  Running  0  72m  cp03
```

Both Ready. Zero restarts since last rollout.

### 25.4 DaemonSets

```
kube-system   cilium         5/5/5/5/5
kube-system   cilium-envoy   5/5/5/5/5
```

**No kube-proxy.**

### 25.5 kube-proxy absence

```bash
kubectl get ds -n kube-system kube-proxy
```

```
Error from server (NotFound): daemonsets.apps "kube-proxy" not found
```

```bash
kubectl get cm -n kube-system kube-proxy
```

```
Error from server (NotFound): configmaps "kube-proxy" not found
```

**kube-proxy fully absent.**

### 25.6 ConfigMap

```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | tail -8
```

**Output:**

```
kubernetesVersion: v1.36.4
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
proxy:
  disabled: true
scheduler: {}
```

### 25.7 Cilium

```
Kubernetes:              Ok         1.36 (v1.36.4)
KubeProxyReplacement:    True
Cilium:                  Ok   1.20.1
Routing:                 Network: Tunnel [geneve]
Cluster health:          5/5 reachable
Controller Status:       18/18 healthy
Modules Health:          Stopped(0) Degraded(0) OK(76)
```

### 25.8 etcd

```
All 3 members: 3.6.8
Same raft term, same raft index, same applied index
No errors
```

### 25.9 What changed

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

### 25.10 What stayed the same

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

## 26. Appendices

### Appendix A — How to take an etcd snapshot before an upgrade

**Not executed during this upgrade** (lab environment, no new state since previous snapshot). **In production, this is non-negotiable.**

**Procedure on any control-plane node:**

```bash
# Create backup directory
sudo mkdir -p /root/etcd-backups

# Take the snapshot
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot.db

# Copy with timestamp
sudo cp /var/lib/etcd/snapshot.db \
  /root/etcd-backups/snapshot-pre-upgrade-$(date +%Y%m%d-%H%M%S).db

# Verify integrity
sudo etcdutl snapshot status \
  /root/etcd-backups/snapshot-pre-upgrade-*.db -w table

# Back up cluster identity
sudo tar -czf \
  /root/etcd-backups/kubernetes-config-pre-upgrade-$(date +%Y%m%d-%H%M%S).tar.gz \
  /etc/kubernetes
```

**Why both are needed:** the etcd snapshot has cluster *state*; the `/etc/kubernetes` tarball has cluster *identity* (CAs, certs). Both are required for full recovery.

### Appendix B — How to upgrade containerd when required

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
# Check current version
containerd --version

# Unhold
sudo apt-mark unhold containerd.io

# Check available versions
apt-cache madison containerd.io

# Install target
sudo apt-get install -y containerd.io=<target-version>

# Re-hold
sudo apt-mark hold containerd.io

# Verify config preserved
sudo grep -E 'SystemdCgroup|sandbox_image' /etc/containerd/config.toml

# Restart
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl is-active containerd

# Verify CRI
sudo ctr plugins ls | grep -E 'cri|grpc'
sudo crictl info

# Verify kubelet reconnects
sudo systemctl status kubelet
```

**Warning:** containerd restart briefly interrupts the CRI on that node. Running pods keep running, but new container operations queue.

**Order for cluster-wide:**
1. One worker at a time (cordon, drain, upgrade, uncordon)
2. Then control planes one at a time
3. Verify cluster health between nodes

### Appendix C — How to verify CNI compatibility before an upgrade

**Not executed as a separate step** — Cilium compatibility was verified during Phase 0 planning.

**Check current CNI version:**

```bash
# For Cilium via Helm
helm list -n kube-system
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium version

# Or from the DaemonSet image
kubectl get ds -n kube-system cilium -o yaml | grep -i image
```

**Check the CNI's documented Kubernetes compatibility matrix:**

For Cilium: https://docs.cilium.io/en/stable/network/kubernetes/compatibility/

**Reference — Cilium compatibility:**

| Cilium | Supported Kubernetes |
|--------|----------------------|
| 1.20 | >=1.33 <1.37 |
| 1.19 | >=1.32 <1.36 |
| 1.18 | >=1.30 <1.34 |

**Our case:** Cilium 1.20.1, target K8s 1.36.4. Within range. No Cilium upgrade needed.

**If CNI needs upgrading:**

1. Upgrade CNI first, **before** the K8s upgrade
2. Verify the CNI still supports the *current* K8s version
3. Verify Cilium status: `KubeProxyReplacement: True`, cluster health OK
4. Then proceed with the Kubernetes upgrade
5. After K8s is upgraded, verify CNI status again

### Appendix D — Troubleshooting command reference

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
# Check if kube-proxy exists
kubectl get ds -A | grep kube-proxy
kubectl get cm -n kube-system | grep kube-proxy

# Check iptables rules on a node
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

### Appendix E — Lessons learned

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

### Appendix F — What we would do differently

1. **Patch the `kubeadm-config` ConfigMap with `proxy.disabled: true` BEFORE the upgrade**, not after.

2. **Add `skipPhases` to the `InitConfiguration`/`JoinConfiguration`**, not `ClusterConfiguration`.

3. **After cp03's `upgrade node`, immediately check for kube-proxy reinstallation.** Don't assume `--skip-phases` on cp01 was sufficient.

4. **Clean iptables proactively** after any kube-proxy removal. Don't wait for a symptom.

5. **Take a fresh etcd snapshot immediately before the upgrade** — even in a lab.

6. **Test DNS with FQDNs**, not short names.

7. **Reboot with `ssh -t`** or `shutdown -r +1`. Avoid the half-reboot problem.

8. **Check installed kernel packages** before spending time on reboots. If the package isn't there, no reboot will activate it.

9. **Document each step's expected output before executing.** This runbook was built as we went; next time we start from it.

### Appendix G — Complete command index

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

**Kernel update (cp01 incident):**

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
