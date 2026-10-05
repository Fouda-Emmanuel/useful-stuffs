# Kubernetes HA — Validation & Failover Runbook

A complete, hands-on runbook for validating and testing High Availability
on a kubeadm-based Kubernetes cluster with stacked etcd and HAProxy.

Written from **real experiments** on a 5-node cluster (3 control-plane +
2 workers). Includes the mistakes, the misconceptions, and the exact
commands that finally proved each HA mechanism.

---

## Target Architecture

```
                    ┌──────────────────────┐
                    │      Bastion         │
                    │   kubectl / helm     │
                    └──────────┬───────────┘
                               │
                               │ TCP 6443
                               ▼
                    ┌──────────────────────┐
                    │  HAProxy + DNS       │
                    │  haproxy.lab:6443    │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
         ┌─────────┐      ┌─────────┐      ┌─────────┐
         │  cp01   │      │  cp02   │      │  cp03   │
         │ etcd    │      │ etcd    │      │ etcd    │
         │apiserver│      │apiserver│      │apiserver│
         │ CM      │      │ CM      │      │ CM      │
         │ sched   │      │ sched   │      │ sched   │
         └─────────┘      └─────────┘      └─────────┘

                    ┌──────────────┐
                    │ worker01/02  │
                    │ kubelet+CNI  │
                    └──────────────┘
```

**Assumptions:**

- kubeadm-initiated cluster with **stacked etcd** (etcd runs as static pods on each control-plane)
- External HAProxy in front of the three API servers
- kubectl and worker kubelets use `haproxy.<domain>:6443`
- Control-plane kubelets use their **local** API server (`127.0.0.1:6443`)

---

## Table of Contents

- [Part 1 — Validating the HA Cluster](#part-1--validating-the-ha-cluster)
  - [1.1 Verifying the etcd Cluster](#11-verifying-the-etcd-cluster)
  - [1.2 Testing Connectivity via the Load Balancer](#12-testing-connectivity-via-the-load-balancer)
  - [1.3 Verifying the kubectl Access Point](#13-verifying-the-kubectl-access-point)
  - [1.4 Deploying a Test Application](#14-deploying-a-test-application)
  - [1.5 Verifying the HAProxy Statistics Interface](#15-verifying-the-haproxy-statistics-interface)
  - [1.6 Verifying System Pod Distribution](#16-verifying-system-pod-distribution)
- [Part 2 — Understanding the HA Mechanisms](#part-2--understanding-the-ha-mechanisms)
  - [2.1 The Three Independent HA Mechanisms](#21-the-three-independent-ha-mechanisms)
  - [2.2 The Static Pod Three-Layer Model](#22-the-static-pod-three-layer-model)
  - [2.3 The Leader-Election Lease](#23-the-leader-election-lease)
- [Part 3 — Failover Testing](#part-3--failover-testing)
  - [3.1 Identifying Current Leaders](#31-identifying-current-leaders)
  - [3.2 Experiment 1 — Node Failure Detection](#32-experiment-1--node-failure-detection)
  - [3.3 Experiment 2 — Controller-Manager Leader Failover](#33-experiment-2--controller-manager-leader-failover)
  - [3.4 Experiment 3 — Scheduler Leader Failover](#34-experiment-3--scheduler-leader-failover)
  - [3.5 Experiment 4 — API Server Failure via HAProxy](#35-experiment-4--api-server-failure-via-haproxy)
- [Part 4 — Common Misconceptions](#part-4--common-misconceptions)
- [Part 5 — Reference Tables](#part-5--reference-tables)
- [Part 6 — Restoring Nodes After Testing](#part-6--restoring-nodes-after-testing)
- [Part 7 — Troubleshooting Commands](#part-7--troubleshooting-commands)

---

# Part 1 — Validating the HA Cluster

Before running any failover test, validate that HA is properly configured.
Each check below answers a specific question about the cluster's HA posture.

---

## 1.1 Verifying the etcd Cluster

### Command

```bash
kubectl get pods -n kube-system -l component=etcd -o wide
```

### Breaking it down

| Part | Meaning |
|------|---------|
| `kubectl` | Query the Kubernetes API (via HAProxy, not etcd directly) |
| `get pods` | List Pod objects |
| `-n kube-system` | In the kube-system namespace (where control-plane components live) |
| `-l component=etcd` | Label selector — only pods labeled `component=etcd` |
| `-o wide` | Show extra columns (IP, NODE) |

### Expected output

```
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE
etcd-cp01.lab           1/1     Running   0          25m   10.10.50.12   cp01.lab
etcd-cp02.lab           1/1     Running   0          20m   10.10.50.13   cp02.lab
etcd-cp03.lab           1/1     Running   0          17m   10.10.50.14   cp03.lab
```

### What this proves

- Three etcd members, one per control-plane node
- All `Running`, `0` restarts
- Stacked etcd topology (etcd inside each CP node, managed by kubelet)

### Why three members

etcd uses **Raft consensus** with a quorum requirement.

| Members | Quorum | Tolerates failure of |
|---------|--------|----------------------|
| 1 | 1 | 0 nodes |
| 2 | 2 | 0 nodes (no real HA) |
| **3** | **2** | **1 node** |
| 5 | 3 | 2 nodes |

**Always use an odd number.** Even numbers don't add fault tolerance and
make quorum math harder.

### What this command does NOT prove

- That etcd members can communicate
- That a Raft leader exists
- That etcd accepts writes
- That latency is healthy

For that, use `etcdctl`. For HA validation, three running pods is enough
because quorum math is what matters.

---

## 1.2 Testing Connectivity via the Load Balancer

### Command

```bash
curl -k https://haproxy.<domain>:6443/readyz
```

### Breaking it down

```
https://haproxy.<domain>:6443/readyz
   │           │             │      │
   │           │             │      └── API health endpoint
   │           │             └── Kubernetes API port
   │           └── HAProxy DNS name
   └── TLS (self-signed → curl -k to skip verification)
```

### Expected output

```
ok
```

### What this proves

The request traveled through the **entire path**:

```
curl → DNS → HAProxy → one of cp01/cp02/cp03 → API server → /readyz
```

If any hop were broken, curl would fail or return a non-200 status.

### Why via HAProxy, not a specific node

- `curl cp01:6443` only proves cp01 works
- `curl haproxy:6443` proves the **HA path** works

### About `-k`

`-k` disables TLS certificate verification. It does **not** disable TLS.
HTTPS is still in use. It just means curl won't reject the self-signed
internal CA.

Fine for quick health checks. **Not** fine for production clients.

### `/readyz` vs `/healthz` vs `/livez`

| Endpoint | Meaning |
|----------|---------|
| `/healthz` | Basic "is the process alive" |
| `/livez` | Liveness — restart-worthy issues |
| `/readyz` | Readiness — deep check (etcd, admission, etc.) |

For HA validation, `/readyz` is the best indicator.

---

## 1.3 Verifying the kubectl Access Point

### Command

```bash
grep server ~/.kube/config
```

### Expected output

```
server: https://haproxy.<domain>:6443
```

### What this proves

Every `kubectl` command goes through HAProxy, not a specific node. This is
the **HA contract** for admin access.

### Why it matters

If kubeconfig said:

```
server: https://cp01.<domain>:6443
```

Then cp01 dying would break all admin access. With HAProxy:

```
                  ┌──► cp01
                  │
kubectl ──► HAProxy├──► cp02
                  │
                  └──► cp03
```

Any single node can fail; kubectl still works.

### Control-plane vs worker kubelets — the distinction

The training and production patterns are:

| Client | Endpoint used | Why |
|--------|--------------|-----|
| Control-plane kubelet | `127.0.0.1:6443` | Local, no extra hop, no dependency on HAProxy |
| Worker kubelet | `haproxy.<domain>:6443` | No local API server |
| kubectl (external) | `haproxy.<domain>:6443` | HA for admin access |

**Local for self, LB for everyone else.**

This avoids a subtle circular dependency: if HAProxy needs the API server
to be reachable, and the API server needs HAProxy... using localhost for
self-management breaks the cycle.

---

## 1.4 Deploying a Test Application

### Command

```bash
kubectl create deployment nginx-ha-test --image=nginx --replicas=3
kubectl get pods -o wide
```

### Expected output

```
NAME                              READY   STATUS    IP           NODE
nginx-ha-test-5d7c5d8f7b-8x9mk    1/1     Running   10.244.1.15  worker01
nginx-ha-test-5d7c5d8f7b-n2k7p    1/1     Running   10.244.2.23  worker02
nginx-ha-test-5d7c5d8f7b-q4t8v    1/1     Running   10.244.1.16  worker01
```

### What this proves

- **API works** (deployment object created)
- **Controller-manager works** (ReplicaSet created 3 Pods)
- **Scheduler works** (Pods assigned to workers)
- **kubelet + CNI works** (containers running, Pod IPs assigned)
- **Load balancing works** (Pods spread across worker01 and worker02)

### The full chain

```
kubectl → API server → etcd
                │
                └─► controller-manager
                          │
                          └─► creates ReplicaSet
                                   │
                                   └─► creates 3 Pods
                                            │
                                            └─► scheduler assigns to workers
                                                     │
                                                     └─► kubelet runs them
                                                              │
                                                              └─► CNI assigns Pod IPs
```

Every control-plane component must be working for this to succeed.

### Cleanup

```bash
kubectl delete deployment nginx-ha-test
```

**Why delete the Deployment, not the Pods:** the Deployment owns a
ReplicaSet, which owns the Pods. Deleting the Deployment removes the
whole hierarchy. Deleting individual Pods would just cause the
ReplicaSet to create replacements.

---

## 1.5 Verifying the HAProxy Statistics Interface

### Access

```
http://haproxy.<domain>:9999/stats
```

### What you see

A real-time dashboard showing:

- Backend servers (`k8s_api_backend`)
- Status per server: `UP` (green) or `DOWN` (red)
- Session counts, traffic counters, health check results

### Expected output

```
k8s_api_backend
  cp01   UP    L4OK in 0ms
  cp02   UP    L4OK in 0ms
  cp03   UP    L4OK in 0ms
```

### Why this matters

HAProxy is doing **TCP health checks** against each control-plane node's
`:6443`. As soon as a node's API server becomes unreachable, HAProxy
marks it `DOWN` and stops routing traffic there.

This is the **observability layer** of your HA. You can literally watch
a node go DOWN in real time during failover.

### Refresh behavior

- Auto-refreshes every 5 seconds
- Manual refresh links available
- CSV / JSON export available

### Health check behavior

HAProxy's check is TCP-level: "Can I connect to `:6443`?"

It does **not** verify TLS, does **not** call `/readyz`, does **not** know
anything about Kubernetes. As long as something is listening on `:6443`,
the backend is `UP`.

This becomes important during testing: killing kubelet doesn't affect
HAProxy; killing the apiserver container does.

---

## 1.6 Verifying System Pod Distribution

### Command

```bash
kubectl get pods -n kube-system -o wide | \
  grep -v NAME | \
  awk '{print $1, $8}' | \
  sort -k2
```

### Breaking down the pipeline

| Stage | What it does |
|-------|--------------|
| `kubectl get pods -n kube-system -o wide` | List all kube-system pods with wide output |
| `grep -v NAME` | Remove the header row |
| `awk '{print $1, $8}'` | Print only pod name (col 1) and node name (col 8) |
| `sort -k2` | Sort by node name |

### Expected distribution

```
Control-plane nodes:
  cp01: etcd, apiserver, controller-manager, scheduler, cilium, [kube-proxy]
  cp02: etcd, apiserver, controller-manager, scheduler, cilium, [kube-proxy]
  cp03: etcd, apiserver, controller-manager, scheduler, cilium, [kube-proxy]

Worker nodes:
  worker01: cilium, [kube-proxy], coredns
  worker02: cilium, [kube-proxy], coredns
```

### Component types and where they run

| Component | Type | Runs on |
|-----------|------|---------|
| etcd | Static pod | All control-plane nodes |
| kube-apiserver | Static pod | All control-plane nodes |
| kube-controller-manager | Static pod | All control-plane nodes |
| kube-scheduler | Static pod | All control-plane nodes |
| CNI (Cilium/Calico) | DaemonSet | All nodes |
| kube-proxy | DaemonSet | All nodes (unless replaced) |
| CoreDNS | Deployment | Usually workers |

### The three controller types, distinguished

| Controller | Manages | Scheduling |
|-----------|---------|------------|
| **Static pod** | Control-plane components | Watched by kubelet from `/etc/kubernetes/manifests/` |
| **DaemonSet** | One pod per node | Managed by kube-controller-manager |
| **Deployment** | N replicas | Managed by kube-controller-manager |

**Static pods are special:** they bypass the scheduler and the API server.
Kubelet watches the manifest directory and runs whatever's declared there.
The API server only sees a "mirror pod" reflection.

---

# Part 2 — Understanding the HA Mechanisms

Before we can test HA, we need to understand **what "HA" means** at each
layer. There are three independent mechanisms, and they trigger on
different failures with different timeouts.

---

## 2.1 The Three Independent HA Mechanisms

### A. Node Lease (kubelet → node health)

```
kubelet
   │  renews every ~10s
   ▼
Lease: kube-node-lease/<node-name>
   │
   │  watched by
   ▼
kube-controller-manager (node controller)
   │
   │  if no renewal for 40s (node-monitor-grace-period)
   ▼
Node.status.conditions[Ready] = False  →  "NotReady"
```

**Answers:** *Is this node reporting healthy?*

**Timeout:** ~40 seconds.

### B. Leader-Election Lease (controller-manager / scheduler HA)

```
controller-manager on cp01  ─┐
controller-manager on cp02  ─┼──►  Lease: kube-system/kube-controller-manager
controller-manager on cp03  ─┘
                                    │
                                    │  current holder renews every ~2s
                                    ▼
                            Only ONE holder at a time.
                            If holder stops renewing for 15s,
                            another instance acquires the lease.
```

**Answers:** *Which instance is actively doing the work?*

**Timeout:** 15 seconds (`leaseDurationSeconds`).

### C. HAProxy TCP health check (API reachability)

```
HAProxy  ──TCP connect──►  <node>:6443
   │
   │  if connect succeeds → backend UP
   │  if connect fails   → backend DOWN
   ▼
client requests routed only to UP backends
```

**Answers:** *Can I reach the API server on this node?*

**Timeout:** typically 10-15s (health check interval × fall count).

### They are independent

A single event can trigger some, all, or none of them:

| Event | Node Lease | Leader Lease | HAProxy |
|-------|-----------|--------------|---------|
| Stop kubelet | ❌ breaks (NotReady) | ✅ still renewing | ✅ still UP |
| Stop apiserver container | ✅ kubelet alive | ✅ still renewing | ❌ breaks (DOWN) |
| Stop CM container | ✅ kubelet alive | ❌ breaks (failover) | ✅ still UP |
| Stop containerd | ❌ breaks | ❌ breaks | ❌ breaks |
| Node poweroff | ❌ breaks | ❌ breaks | ❌ breaks |

**Understanding this table is the key to understanding Kubernetes HA.**

---

## 2.2 The Static Pod Three-Layer Model

Every control-plane component (etcd, apiserver, controller-manager,
scheduler) runs as a **static pod**. Static pods exist in **three
independent layers**:

```
┌────────────────────────────────────────────────────────┐
│  Layer 1 — Manifest file (on node disk)                │
│                                                        │
│  /etc/kubernetes/manifests/<component>.yaml            │
│                                                        │
│  Source of truth. Only kubelet reads it.               │
└────────────────────────────────────────────────────────┘
                        ↓  read by kubelet
┌────────────────────────────────────────────────────────┐
│  Layer 2 — Container (in containerd)                   │
│                                                        │
│  The actual running process.                           │
│  THIS is what renews the leader-election Lease.        │
└────────────────────────────────────────────────────────┘
                        ↓  registered by kubelet
┌────────────────────────────────────────────────────────┐
│  Layer 3 — Mirror pod (in API server / etcd)           │
│                                                        │
│  A read-only reflection object.                        │
│  kubectl shows you THIS. It is NOT the pod.            │
└────────────────────────────────────────────────────────┘
```

### Critical implications

| Action | Effect on container | Effect on Lease |
|--------|---------------------|-----------------|
| `kubectl delete pod` (mirror) | ❌ No effect | ❌ No effect |
| `systemctl stop kubelet` | ❌ No effect | ❌ No effect |
| `crictl stop <id>` (kubelet running) | ⚠️ Restarts in ~2s | ❌ No effect |
| `systemctl stop kubelet` + `crictl stop <id>` | ✅ Container stays dead | ✅ Lease expires → failover |

**The only way to reliably trigger failover of a static-pod-based
component is to stop kubelet first (so it can't reconcile), and then
stop the container (so the process actually dies).**

We'll prove this in Part 3.

---

## 2.3 The Leader-Election Lease

Kubernetes uses a `Lease` object (kind: `Lease`, API group
`coordination.k8s.io/v1`) to coordinate leader election among
controller-manager and scheduler instances.

### Fields that matter

```yaml
spec:
  holderIdentity:       "cp02.lab_af203935-8c81-..."  # who holds it
  leaseDurationSeconds: 15                             # valid for 15s
  acquireTime:          "2026-10-05T18:04:36.917Z"     # when acquired
  renewTime:            "2026-10-05T19:05:28.698Z"     # last renewal
  leaseTransitions:     3                              # change count
```

| Field | Meaning | Diagnostic use |
|-------|---------|----------------|
| `holderIdentity` | Which instance is the leader | `<node>_<uuid>` |
| `renewTime` | Last time holder renewed | Stale (>15s) → leader dead |
| `acquireTime` | When holder took over | Recent → new failover |
| `leaseDurationSeconds` | Lease TTL | Failover delay = this value |
| `leaseTransitions` | Lifetime change count | Increments on failover |

### The `holderIdentity` format

```
cp02.lab_af203935-8c81-4232-8871-3620d9844b26
│         │
│         └── Unique instance UUID (changes on every container restart)
└── Node name
```

**Two things can change here:**

- Node name stays, UUID changes → container restarted, same node kept leadership
- Node name changes → real failover to a different node

### How leadership is defended

The leader:

1. Holds the Lease
2. Renews it every ~2 seconds (well under `leaseDurationSeconds: 15`)
3. Continues to do work (schedule pods, reconcile controllers)

The standbys:

1. Watch the Lease
2. If `renewTime` is older than `leaseDurationSeconds` → the lease is considered expired
3. Standbys race to acquire it (`holderIdentity` → their node name)
4. Winner becomes leader, `leaseTransitions` increments

### Reading a lease quickly

```bash
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}transitions={.spec.leaseTransitions}{"\n"}renew={.spec.renewTime}{"\n"}'
```

---

# Part 3 — Failover Testing

Now we deliberately break things and observe HA recovery.

---

## 3.1 Identifying Current Leaders

### Commands

```bash
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}transitions={.spec.leaseTransitions}{"\n"}'

kubectl get lease -n kube-system kube-scheduler \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}transitions={.spec.leaseTransitions}{"\n"}'
```

### Example output

```
holder=cp02.lab_af203935-8c81-4232-8871-3620d9844b26
transitions=2

holder=cp01.lab_02b521d1-f8b7-48fc-9178-cb907918c56d
transitions=2
```

### Interpretation

| Component | Leader | Note |
|-----------|--------|------|
| kube-controller-manager | cp02 | |
| kube-scheduler | cp01 | |

**Before the test:** note these values. Record `holderIdentity` and
`leaseTransitions` for both.

### Why this matters

- If we kill cp01, **only the scheduler** should failover
- Controller-manager should stay on cp02 (unaffected)
- If we kill cp02 instead, the reverse happens

**This makes the experiment clean:** one change → one observation.

### Three different "leader" concepts — don't confuse them

| Mechanism | What it means | Where to see it |
|-----------|--------------|-----------------|
| **etcd Raft** | Internal consensus leader | `etcdctl endpoint status` |
| **controller-manager Lease** | Active CM instance | `kubectl get lease ...kube-controller-manager` |
| **scheduler Lease** | Active scheduler instance | `kubectl get lease ...kube-scheduler` |

**These are three separate leaders.** A node can hold any, all, or none.

---

## 3.2 Experiment 1 — Node Failure Detection

**Goal:** Observe how Kubernetes detects that a node's kubelet has stopped.

**Mechanism tested:** Node Lease.

### Step 1 — Watch the Node Lease and Node Status

Open a **second terminal on bastion** and run:

```bash
while true; do
  echo "=== $(date -u +%H:%M:%S) ==="
  kubectl get lease -n kube-node-lease <NODE> \
    -o jsonpath='  lease-renew={.spec.renewTime}{"\n"}'
  kubectl get node <NODE> \
    -o jsonpath='  node-status={.status.conditions[?(@.type=="Ready")].status}{"\n"}'
  sleep 3
done
```

Replace `<NODE>` with the target node (e.g. `cp02.lab`).

### Step 2 — Stop kubelet on the target node

```bash
ssh ubuntu@<NODE>
sudo systemctl stop kubelet
sudo systemctl status kubelet --no-pager
```

### Expected timeline

| Time | Event |
|------|-------|
| t+0s | kubelet stops |
| t+~10s | Node Lease `renewTime` freezes |
| t+~40s | `node-status` changes `True` → `False` |

### What does NOT happen (all at once)

- Containers on the node **keep running** (containerd still running, kubelet wasn't their owner)
- Controller-manager / scheduler on this node **keep running** and **keep renewing their leases** if they were the leader
- HAProxy still shows the node `UP` (apiserver still on :6443)
- Mirror pods in `kubectl get pods -n kube-system` **still show `Running`** (stale)

### Verify

```bash
# Node status
kubectl get nodes
# Target: NotReady. Others: Ready.

# Mirror pods (STALE)
kubectl get pods -n kube-system -o wide | grep <NODE>
# Still shows Running. This is a frozen snapshot.

# Ground truth: containers on the node
ssh ubuntu@<NODE> 'sudo crictl ps'
# You will see etcd, apiserver, controller-manager, scheduler ALL still Running

# Controller-manager lease (if leader was on this node)
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}renew={.spec.renewTime}{"\n"}'
# Holder still cp02. renewTime still advancing. No failover.
```

### The four simultaneous truths

| Source | Says about the node |
|--------|---------------------|
| `kubectl get nodes` | NotReady |
| Mirror pods | Running (stale) |
| `crictl ps` on the node | Running (truth) |
| HAProxy | UP |
| Leader-election Lease | Still held by this node |

**All correct. None conflicting.**

### Restore

```bash
ssh ubuntu@<NODE>
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
```

Within ~40s, node returns to `Ready`.

### Key lesson

> **Stopping kubelet does NOT stop the containers it manages. It only
> stops the reconciliation loop and the Node Lease heartbeat.**

---

## 3.3 Experiment 2 — Controller-Manager Leader Failover

**Goal:** Trigger and observe a real controller-manager leader failover.

**Mechanism tested:** Leader-election Lease.

### Step 1 — Identify the current leader

```bash
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}transitions={.spec.leaseTransitions}{"\n"}renew={.spec.renewTime}{"\n"}'
```

Note the **node name** in `holderIdentity`. This is the node we break.

### Step 2 — Watch the Lease in real time

Open a second terminal on bastion:

```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml -w
```

Keep it running.

### Step 3 — The wrong ways (see that they don't work)

**Attempt A — delete the mirror pod:**

```bash
kubectl delete pod -n kube-system kube-controller-manager-<leader-node>
```

**Result:** Pod object goes `Terminating` → disappears. But the container
keeps running. Lease keeps renewing. **No failover.**

**Attempt B — force delete:**

```bash
kubectl delete --force pod -n kube-system kube-controller-manager-<leader-node>
```

**Result:** Same. The mirror pod record is gone, but the container keeps
running. **No failover.**

**Attempt C — stop kubelet only:**

```bash
ssh ubuntu@<leader-node>
sudo systemctl stop kubelet
```

**Result:** Node goes `NotReady` after ~40s. But the CM container keeps
running and renewing the lease. **No failover.**

**Attempt D — stop the container while kubelet is still running:**

```bash
ssh ubuntu@<leader-node>
sudo crictl ps --name kube-controller-manager
# note the CONTAINER ID (first column)
sudo crictl stop <container-id>
```

**Result:** Container restarts in ~2s (kubelet reconciles from the
manifest). The lease never even expires. **No failover.**

### Step 4 — The correct way — stop kubelet AND the container

```bash
ssh ubuntu@<leader-node>

# 1. Stop kubelet so it won't reconcile the container back
sudo systemctl stop kubelet
sudo systemctl is-active kubelet
# → inactive

# 2. Stop the CM container
sudo crictl ps --name kube-controller-manager
# note the CONTAINER ID
sudo crictl stop <container-id>

# 3. Verify it stays stopped
sleep 5
sudo crictl ps | grep kube-controller-manager || echo "STOPPED"
```

### Step 5 — Watch the failover

On the bastion terminal running `kubectl get lease ... -w`:

| Time | What you see |
|------|-------------|
| t+0s | Container stopped. `renewTime` freezes. |
| t+0 to t+15s | Lease stays with old holder. `renewTime` stale. |
| t+~15s | `holderIdentity` changes to a new node. |
| t+~15s | `acquireTime` updates. |
| t+~15s | `leaseTransitions` increments by 1. |

### Step 6 — Verify

```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml
# holderIdentity, acquireTime, leaseTransitions all changed
```

### Step 7 — Prove the cluster still works

```bash
curl -k https://haproxy.<domain>:6443/readyz
# → ok

kubectl get nodes
# Target NotReady, others Ready

kubectl create deployment cm-failover-test --image=nginx --replicas=2
kubectl get pods -o wide
# 2 pods Running on worker nodes

kubectl delete deployment cm-failover-test
```

### Step 8 — Restore

```bash
ssh ubuntu@<leader-node>
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
sudo crictl ps | grep kube-controller-manager
# Container back (recreated from manifest)
```

**Note:** the lease does **NOT** move back to the original node.
Leadership follows health, not ownership. `leaseTransitions` stays
incremented.

### Key lessons

- Leader-election Lease is renewed by the **container process**.
- To force failover, kill the process, not the pod object or the kubelet.
- Failover timing = `leaseDurationSeconds` (15s).
- Leadership does **not** "return home" after recovery.

---

## 3.4 Experiment 3 — Scheduler Leader Failover

Identical procedure to Experiment 2, but for `kube-scheduler`.

### Step 1 — Identify the current scheduler leader

```bash
kubectl get lease -n kube-system kube-scheduler \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}transitions={.spec.leaseTransitions}{"\n"}renew={.spec.renewTime}{"\n"}'
```

### Step 2 — Watch

```bash
kubectl get lease -n kube-system kube-scheduler -o yaml -w
```

### Step 3 — On the leader node

```bash
sudo systemctl stop kubelet
sudo crictl ps --name kube-scheduler
# note container ID
sudo crictl stop <container-id>
sleep 5
sudo crictl ps | grep kube-scheduler || echo "STOPPED"
```

### Step 4 — Watch the failover on bastion

Same behavior as Experiment 2. `leaseTransitions` increments,
`holderIdentity` changes to a new node.

### Step 5 — Prove the new scheduler schedules

```bash
kubectl create deployment sched-failover-test --image=nginx --replicas=2
kubectl get pods -o wide
# Pods scheduled and Running

kubectl delete deployment sched-failover-test
```

### Step 6 — Restore

```bash
ssh ubuntu@<leader-node>
sudo systemctl start kubelet
```

### Key lessons

Scheduler and controller-manager use the **same** Lease-based
leader-election pattern. Different lease names, same mechanism.

---

## 3.5 Experiment 4 — API Server Failure via HAProxy

**Goal:** Observe HAProxy detecting a dead API server and removing it.

**Mechanism tested:** HAProxy TCP health check.

### Step 1 — Watch HAProxy stats

Open in browser:

```
http://haproxy.<domain>:9999/stats
```

All backends should show `UP` in green.

### Step 2 — Kill the API server container

Unlike the previous experiments, we want the **whole node** to look dead
from HAProxy's perspective.

```bash
ssh ubuntu@<target-node>

# Stop kubelet to prevent reconciliation
sudo systemctl stop kubelet

# Stop the API server container
sudo crictl ps --name kube-apiserver
# note container ID
sudo crictl stop <container-id>
```

Or, to kill everything at once:

```bash
sudo systemctl stop kubelet
sudo systemctl stop containerd
```

### Step 3 — Watch the HAProxy dashboard

| Time | Event |
|------|-------|
| t+0s | API server process dies |
| t+~10-15s | Backend marked `DOWN` (red) |
| subsequent | Client requests routed to remaining backends |

### Step 4 — Verify

```bash
# API still reachable (via other backends)
curl -k https://haproxy.<domain>:6443/readyz
# → ok

kubectl get nodes
# Target NotReady

kubectl create deployment apiserver-down-test --image=nginx --replicas=2
kubectl get pods -o wide
# Still works

kubectl delete deployment apiserver-down-test
```

### Step 5 — Restore

```bash
ssh ubuntu@<target-node>
sudo systemctl start containerd   # if you stopped it
sudo systemctl start kubelet
```

Wait ~30-60s. Backend marked `UP` again in HAProxy.

### Key lessons

- HAProxy doesn't care about Kubernetes. It does a TCP connect to `:6443`.
- As long as something is listening, the backend is `UP`.
- Only when the process dies does HAProxy remove the backend.

---

# Part 4 — Common Misconceptions

These are the mistakes we actually made. Read carefully.

### ❌ "Stopping kubelet will stop the containers it manages"

**Wrong.** kubelet is a **reconciler**, not the process owner. It
creates containers via containerd and watches them. Stopping kubelet
stops the reconciliation loop and the Node Lease heartbeat, but
containerd keeps running the containers.

**Correct primitive:** to actually stop a container, use
`crictl stop <container-id>`.

### ❌ "Deleting the pod object will kill the container"

**Wrong** for static pods. The pod object you see with `kubectl get pods`
is a **mirror pod** — a read-only reflection. Deleting it removes the
API server's record but does nothing to the container.

For **normal pods** (owned by Deployment/ReplicaSet), this is true.

For **static pods**, the manifest on disk is the source of truth, and
only kubelet reads it.

### ❌ "Restarting kubelet will bring back the pod"

**Correct — but only if the container is dead.** kubelet reads the
manifest and reconciles. If the container is missing, it creates one.
If already running, it adopts it.

That's why stopping the container while kubelet is running brings it
back in ~2s — and why you must stop kubelet first to observe failover.

### ❌ "NotReady means everything on the node is dead"

**Wrong.** `NotReady` is determined by the Node Lease. Only kubelet
renews it. If kubelet stops, the node is `NotReady` — but every other
process (etcd, apiserver, controller-manager, scheduler, CNI) can still
be running.

You may have:

- `kubectl get nodes` → NotReady
- `kubectl get pods -n kube-system` → Running (stale)
- `crictl ps` on the node → containers running
- HAProxy → backend UP
- Controller-manager Lease → still held by this node

**All true simultaneously.**

### ❌ "The mirror pod status is authoritative"

**Wrong on a dead-kubelet node.** Mirror pod status is updated by
kubelet. If kubelet is dead, the mirror pod is a frozen snapshot.
kubectl will show `Running` indefinitely.

Always cross-check with:

- `kubectl get nodes`
- Node Lease `renewTime`
- `crictl ps` on the node

### ❌ "Leader failover happens instantly"

**Wrong.** Failover timing = `leaseDurationSeconds` (15s default).

Related timeouts:

- Node `NotReady`: `node-monitor-grace-period` (40s)
- HAProxy backend DOWN: `inter × fall` (typically 10-15s)

**Each mechanism has its own timeout.**

### ❌ "Leadership returns to the original node after recovery"

**Wrong.** Leadership follows health. Once a new leader acquires the
lease, it keeps it until it fails. The original node's component runs
as a **standby** when it recovers.

`leaseTransitions` does **not** decrement on recovery.

---

# Part 5 — Reference Tables

## 5.1 Action vs Outcome

| Action | Container process | Node Lease renew | CM/Sched Lease renew | HAProxy backend | Node status | Failover? |
|--------|-------------------|------------------|----------------------|-----------------|-------------|-----------|
| `kubectl delete pod` (mirror) | ✅ running | ✅ | ✅ | ✅ UP | ✅ Ready | ❌ |
| `systemctl stop kubelet` | ✅ running | ❌ stops | ✅ (still renewing) | ✅ UP | ❌ NotReady | ❌ |
| `crictl stop` (kubelet running) | ⚠️ restarts in ~2s | ✅ | ✅ | ✅ UP | ✅ Ready | ❌ |
| `systemctl stop kubelet` + `crictl stop` | ❌ dead | ❌ stops | ❌ expires 15s | ✅ UP (apiserver alive) | ❌ NotReady | ✅ |
| `systemctl stop containerd` | ❌ all dead | ❌ stops | ❌ expires | ❌ DOWN | ❌ NotReady | ✅ |
| Node poweroff / VM shutdown | ❌ all dead | ❌ stops | ❌ expires | ❌ DOWN | ❌ NotReady | ✅ |

## 5.2 Error / Symptom vs Layer

| Symptom | Layer | Common cause |
|---------|-------|--------------|
| DNS resolution failed | DNS | Wrong name, no record, resolver unreachable |
| `no route to host` | L3 | Missing route, wrong subnet |
| `i/o timeout` | L3/L4 firewall | Security Group, iptables, NACL |
| `connection refused` | L4 | Nothing listening, wrong port |
| `connection reset` | L4 | TCP RST (stateful firewall) |
| `x509: certificate is valid for X, not Y` | TLS | SAN mismatch |
| `tls: handshake failure` | TLS | Protocol/cipher mismatch |
| `401 Unauthorized` | HTTP/API | Bad token, expired cert |
| `403 Forbidden` | HTTP/API | RBAC, admission webhook |
| `kubeadm join` stuck at preflight | Network/SG | Cannot reach HAProxy on 6443 |

## 5.3 Lease Field Reference

| Field | Meaning | Diagnostic |
|-------|---------|------------|
| `holderIdentity` | Who holds the lease | `<node>_<uuid>` |
| `renewTime` | Last renewal | Stale (>15s) → leader dead |
| `acquireTime` | When leader took over | Recent → new failover |
| `leaseDurationSeconds` | Lease TTL | Failover delay |
| `leaseTransitions` | Lifetime change count | Increments on failover |

## 5.4 Static Pod vs Normal Pod

| Aspect | Static pod | Normal pod |
|--------|-----------|------------|
| Source of truth | Manifest on disk | API server / etcd |
| Managed by | kubelet | Controller (Deployment, DaemonSet, etc.) |
| Visible via `kubectl get pods` | Yes (as "mirror pod") | Yes |
| Deletable via `kubectl delete pod` | ❌ No | ✅ Yes |
| Restarts on failure | kubelet recreates from manifest | Controller recreates |
| Owner | `Node/<node-name>` | Whatever owns it |
| Example | etcd, apiserver, CM, scheduler | CoreDNS, Cilium, user workloads |

---

# Part 6 — Restoring Nodes After Testing

### If you stopped kubelet only

```bash
ssh ubuntu@<node>
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
```

Node returns to `Ready` in ~40s. Containers were never affected.

### If you stopped kubelet + container

```bash
ssh ubuntu@<node>
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
sudo crictl ps | grep <component>
# kubelet recreates the container from the manifest
```

### If you stopped containerd

```bash
ssh ubuntu@<node>
sudo systemctl start containerd
sudo systemctl status containerd --no-pager
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
```

### Verification from bastion

```bash
kubectl get nodes
kubectl get pods -n kube-system -o wide
kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml
curl -k https://haproxy.<domain>:6443/readyz
```

**Note:** leases will **not** move back to the recovered node.
Correct behavior.

---

# Part 7 — Troubleshooting Commands

### Node / kubelet health

```bash
kubectl get nodes -o wide
kubectl get node <node> -o yaml | grep -A5 conditions
kubectl get lease -n kube-node-lease <node> -o yaml
```

### Leader election state

```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml
```

### Watch lease changes

```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml -w
```

### Container-level ground truth (on the node)

```bash
sudo crictl ps
sudo crictl ps -a
sudo crictl inspect <container-id>
sudo crictl logs <container-id>
```

### Stop a static pod container

```bash
sudo crictl ps --name <component>
# get the container ID
sudo crictl stop <container-id>
```

### Static pod manifests

```bash
ls /etc/kubernetes/manifests/
sudo cat /etc/kubernetes/manifests/<component>.yaml
```

### API server health

```bash
curl -k https://haproxy.<domain>:6443/readyz
curl -k https://haproxy.<domain>:6443/healthz
curl -k https://haproxy.<domain>:6443/livez
```

### System pods

```bash
kubectl get pods -n kube-system -o wide
```

### Node Lease (kubelet heartbeat)

```bash
kubectl get lease -n kube-node-lease <node> \
  -o jsonpath='renew={.spec.renewTime}{"\n"}'
```

### Full node description

```bash
kubectl describe node <node>
```

### Check systemd status of kubelet / containerd

```bash
sudo systemctl status kubelet --no-pager
sudo systemctl status containerd --no-pager
```

### Check kubelet logs

```bash
sudo journalctl -u kubelet -n 50 --no-pager
sudo journalctl -u kubelet -f
```

---

## Appendix — The Mental Model, in One Page

```
STATIC POD (e.g. controller-manager)
├── Manifest file     →  /etc/kubernetes/manifests/<component>.yaml
│                         (source of truth, only kubelet reads it)
│
├── Container         →  contains the ACTUAL PROCESS
│                         (this is what renews the leader-election Lease)
│
└── Mirror pod        →  a REFLECTION visible via kubectl
                          (only a description, not the pod)

To trigger leader failover:
  1. Stop kubelet   (so it doesn't reconcile the container back)
  2. Stop container (so the process actually dies)
  3. Wait 15s       (leaseDurationSeconds)
  4. Observe new leader in Lease holderIdentity + leaseTransitions++

To trigger node NotReady:
  - Stop kubelet only
  - Wait 40s (node-monitor-grace-period)

To trigger HAProxy backend DOWN:
  - Kill the apiserver container (or stop containerd)
  - Wait 10-15s (health check interval × fall count)

Three mechanisms. Three timeouts. Three independent signals.
All can be true simultaneously. None of them is "the truth".
```
