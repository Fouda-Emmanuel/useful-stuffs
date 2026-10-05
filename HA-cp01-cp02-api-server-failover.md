# Kubernetes HA — Failover Experiments & Findings (Cp01 & Cp02 Scheduler, Controller-Manager, API Server Test)
A record of hands-on experiments on a live kubeadm HA cluster, the
questions that drove them, and what each experiment revealed.

**Cluster:**

- 3 control-plane nodes: cp01, cp02, cp03 (stacked etcd, static pods)
- 2 workers: worker1, worker2
- HAProxy at `haproxy.<domain>:6443` in front of all three API servers
- Kubernetes v1.35.8, containerd, cilium (eBPF, kube-proxy-free)

---

## Table of Contents

**Foundations**

1. [The Driving Questions](#1-the-driving-questions)
2. [The Four Independent HA Mechanisms](#2-the-four-independent-ha-mechanisms)
3. [What Is a Lease? — The Core Concept](#3-what-is-a-lease--the-core-concept)
4. [etcd Quorum vs CM/Scheduler Leader Election — The Big Distinction](#4-etcd-quorum-vs-cmscheduler-leader-election--the-big-distinction)

**Experiments**

5. [Experiment Set A — Understanding Leader Election](#5-experiment-set-a--understanding-leader-election)
6. [Experiment Set B — Stopping cp01 and cp02 (apiserver included)](#6-experiment-set-b--stopping-cp01-and-cp02-apiserver-included)
7. [Experiment Set C — Making HAProxy Show a Backend as DOWN](#7-experiment-set-c--making-haproxy-show-a-backend-as-down)
8. [Experiment Set D — etcd Quorum Loss](#8-experiment-set-d--etcd-quorum-loss)
9. [Experiment Set E — Apiserver Crash-Loop from Quorum Loss](#9-experiment-set-e--apiserver-crash-loop-from-quorum-loss)

**Reference**

10. [Findings Reference Tables](#10-findings-reference-tables)
11. [The Complete HA Mental Model](#11-the-complete-ha-mental-model)
12. [Recovery Procedures Used](#12-recovery-procedures-used)

---

# FOUNDATIONS

---

## 1. The Driving Questions

These came up during HA validation and drove all the tests below.

### Question 1 — Does leader election follow quorum like etcd?

- Does controller-manager / scheduler use quorum?
- How is that different from etcd?
- Can we tolerate the same number of failures?

### Question 2 — What happens if we stop both cp01 and cp02?

- Does the cluster survive?
- What about etcd quorum?
- Does the answer depend on *what* we stop?

### Question 3 — Why did HAProxy never show any CP as DOWN?

- We stopped kubelet, CM, scheduler, deleted pod objects. HAProxy still
  showed UP.
- What actually triggers HAProxy to mark a backend DOWN?
- What is the difference between "Kubernetes healthy" and "HAProxy healthy"?

### Question 4 (implicit) — What if we stop etcd too?

- Does the cluster finally break?
- At which threshold?
- What happens to existing workloads?

### Question 5 (observed) — Why did cp03's apiserver disappear on its own?

- We didn't stop it. Yet it entered a crash-loop.
- What causes an apiserver to fail to start?
- How does the cluster recover from this?

---

## 2. The Four Independent HA Mechanisms

Kubernetes HA is **not one mechanism** — it's four independent mechanisms,
each with its own trigger, timeout, and consequence. Confusing them is the
#1 source of "surprising" results during testing.

| Mechanism | What it watches | Timeout | Consequence when it fires |
|-----------|----------------|---------|---------------------------|
| **etcd quorum** | Member majority | ~1s (Raft) | Cluster becomes read-only |
| **Node Lease** | kubelet heartbeat | ~40s | Node marked `NotReady` |
| **Leader-election Lease** | CM / scheduler lease renewal | ~15s | New leader elected |
| **HAProxy health check** | TCP connect to `:6443` | ~10-15s | Backend marked `DOWN` |

**Each is triggered by a different failure.** You can fire one, several,
or none of them depending on what you break.

| Event | etcd quorum | Node Lease | Leader Lease | HAProxy |
|-------|:-----------:|:----------:|:------------:|:-------:|
| Stop kubelet on a node | ✅ fine | ❌ breaks | ✅ fine | ✅ UP |
| Stop apiserver container | ✅ fine | ✅ fine | ✅ fine | ❌ DOWN |
| Stop CM container | ✅ fine | ✅ fine | ❌ breaks | ✅ UP |
| Stop etcd on 1 of 3 | ✅ fine (quorum 2/3) | ✅ fine | ✅ fine | ✅ UP (if apiserver alive) |
| Stop etcd on 2 of 3 | ❌ breaks | ❌ breaks | ❌ breaks | ✅ UP (still sees port) |
| Node poweroff | ❌ breaks | ❌ breaks | ❌ breaks | ❌ DOWN |

**This table is the *Rosetta Stone* of Kubernetes HA.** Every "surprising"
result in our experiments is explained by it. Bookmark it — you will
consult it many times.

---

## 3. What Is a Lease? — The Core Concept

Before going further, we need to understand what a **Lease** actually is,
because two of the four HA mechanisms above are lease-based.

### 3.1 The plain-English definition

> **A Lease is a **time-limited lock**. One holder at a time. The holder
> must periodically **renew** it. If the holder stops renewing, the lease
> **expires**, and someone else can take it.**

That's it. Everything else is detail.

### 3.2 Why a lease?

You have three controller-manager instances. You want only **one** of
them to actually do work at a time (create ReplicaSets, reconcile
Deployments, etc.). You need a way to say:

> "Whoever holds this lock, do the work. Everyone else, stay quiet."

A lease solves this. It's mutual exclusion, with automatic failover:

- **Mutual exclusion** → only one holder at a time
- **Automatic failover** → if the holder dies, the lock becomes free
  automatically

### 3.3 The lifecycle of a lease

```
┌─────────────────────────────────────────────────────────────┐
│                     LEASE LIFECYCLE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. ACQUIRE                                                 │
│     Instance A claims the lease:                            │
│       holderIdentity = A                                    │
│       acquireTime    = now                                  │
│                                                             │
│  2. RENEW (every ~2 seconds)                                │
│     Instance A keeps saying "I'm still here":               │
│       renewTime = now                                       │
│                                                             │
│  3. EXPIRY                                                  │
│     If Instance A stops renewing:                           │
│       now - renewTime > leaseDurationSeconds                │
│     The lease is considered free.                           │
│                                                             │
│  4. RE-ACQUIRE                                              │
│     Instance B sees the lease is free, claims it:           │
│       holderIdentity = B                                    │
│       leaseTransitions++                                    │
│                                                             │
│  5. REPEAT                                                  │
│     Instance B renews. Life goes on.                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 What a Lease looks like in Kubernetes

A Kubernetes Lease is an object of kind `Lease` in the
`coordination.k8s.io/v1` API group. You can see it with `kubectl`:

```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml
```

Output (trimmed):

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: cp02.faekcorp.lab_af203935-8c81-4232-8871-3620d9844b26
  leaseDurationSeconds: 15
  acquireTime: "2026-10-05T18:04:36.917Z"
  renewTime: "2026-10-05T19:05:28.698Z"
  leaseTransitions: 3
```

### 3.5 Field-by-field meaning

| Field | Meaning | Why it matters |
|-------|---------|----------------|
| `holderIdentity` | `<node>_<instance-uuid>` — who currently holds the lease | Tells you which instance is active |
| `leaseDurationSeconds` | How long the lease is valid (default 15s) | Determines failover time |
| `acquireTime` | When the current holder took over | Recent value → just failed over |
| `renewTime` | Last time the holder renewed | Stale value (>15s old) → holder may be dead |
| `leaseTransitions` | Lifetime count of leadership changes | Increments on every failover |

### 3.6 The holder identity format

```
cp02.faekcorp.lab_af203935-8c81-4232-8871-3620d9844b26
└──── node ─────┘ └────────── instance UUID ───────────┘
```

- **Node changes** → real failover to another node
- **Only UUID changes** → same node kept leadership, but pod restarted

### 3.7 Why leases are stored in etcd

**This is the crucial subtlety.** The lease is itself a Kubernetes object
— it lives in etcd, next to every other piece of cluster state.

```
controller-manager on cp02
        │
        │  "I want to renew my lease"
        ▼
kube-apiserver
        │
        │  writes the lease update
        ▼
etcd (needs quorum)
```

**So to renew its lease, the leader must write to etcd.** And to write
to etcd, etcd needs quorum.

This is why CM/scheduler **depend on etcd quorum** even though their own
election mechanism doesn't need quorum. We'll come back to this in
Section 4.

### 3.8 Node Lease vs Leader-Election Lease

There are actually **two different kinds of leases** in a Kubernetes
cluster. Don't confuse them.

| Lease | Location | Holder | Renewed by | Purpose |
|-------|----------|--------|------------|---------|
| **Node Lease** | `kube-node-lease/<node>` | The node itself | kubelet | Prove kubelet is alive |
| **Leader-Election Lease** | `kube-system/kube-controller-manager` etc. | Active instance | CM/scheduler pod | Prove leadership is active |

**Node Lease:**

- Renewed every ~10s by kubelet
- If not renewed for 40s, node marked `NotReady`
- **This is what makes `kubectl get nodes` show `NotReady`**

**Leader-Election Lease:**

- Renewed every ~2s by the active CM/scheduler
- If not renewed for 15s, another instance takes over
- **This is what makes controller-manager/scheduler leadership move**

**Two independent lease mechanisms. Different purposes. Different timeouts.**

### 3.9 The mental model

Think of a lease like a **parking spot** with a time limit:

- You park (acquire)
- You keep feeding the meter (renew)
- If you stop feeding, the spot becomes free (expire)
- Someone else parks (re-acquire)

**Nobody is in charge of deciding who parks.** There's no referee.
Everyone just watches the spot. If it's occupied, wait. If it's free,
take it. This distributed-race design is why the system is robust: no
single point of coordination failure.

### 3.10 Lease command cheat sheet

```bash
# Read the current leader-election state
kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml

# Compact view — just the essentials
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='holder={.spec.holderIdentity}{"\n"}transitions={.spec.leaseTransitions}{"\n"}renew={.spec.renewTime}{"\n"}'

# Watch leadership change in real time
kubectl get lease -n kube-system kube-controller-manager -o yaml -w

# Check a node's health lease
kubectl get lease -n kube-node-lease <node-name> -o yaml

# List all leases in a namespace
kubectl get leases -n kube-system
```

### 3.11 Two questions that will help you at 3 AM

**"Is the leader alive?"**

```bash
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='renew={.spec.renewTime}{"\n"}holder={.spec.holderIdentity}{"\n"}'
```

If `renewTime` is more than ~15s old → **leader is dead or partitioned.**
Failover should be happening (or should already have happened).

**"Did a failover just happen?"**

```bash
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='transitions={.spec.leaseTransitions}{"\n"}acquire={.spec.acquireTime}{"\n"}'
```

If `acquireTime` is very recent → **failover just occurred.** Compare
`transitions` to its previous value to confirm.

---

## 4. etcd Quorum vs CM/Scheduler Leader Election — The Big Distinction

This is the single most important conceptual distinction in Kubernetes
HA. It took us several experiments to fully internalize.

### 4.1 etcd — Raft consensus with quorum

etcd stores **cluster state**. Every write must be replicated to a
**majority** of members before being committed.

```
3-member etcd cluster
         │
         │  every write:
         ▼
   leader proposes
         │
   ┌─────┼─────┐
   ▼     ▼     ▼
  cp01  cp02  cp03
   ✅    ✅    ✅
         │
         │  needs ≥2 acks to commit
         ▼
      committed
```

| Members alive | Quorum | Writes possible? |
|---------------|--------|-----------------|
| 3 of 3 | ✅ | ✅ |
| 2 of 3 | ✅ | ✅ |
| 1 of 3 | ❌ | ❌ |
| 0 of 3 | ❌ | ❌ |

**Why quorum:** to prevent split-brain. If two partitions both think
they're the leader, they could diverge. Quorum ensures only one partition
(majority) can commit.

**Failure mode:** *loss of quorum* — the cluster stops accepting writes.
In practice, reads may also fail because authorization checks require
etcd consistency, and the apiserver itself may fail to start.

### 4.2 Controller-manager and scheduler — Lease-based leader election

These components are **not** a distributed database. They don't need
consensus. They need a **single-writer lock**.

```
3 controller-manager instances
         │
         │  all try to acquire a single Lease
         ▼
   kube-system/kube-controller-manager
         │
   ┌─────┼─────┐
   ▼     ▼     ▼
  cp01  cp02  cp03
   ✗     ✅    ✗
         │
         │  cp02 wins, becomes leader
         │  renews lease every ~2s
         ▼
      leader = cp02
```

| Instances alive | Need majority? | Leader possible? |
|-----------------|----------------|------------------|
| 3 of 3 | ❌ No | ✅ |
| 2 of 3 | ❌ No | ✅ |
| 1 of 3 | ❌ No | ✅ (the lone one leads) |
| 0 of 3 | ❌ No | ❌ (no instance left) |

**Why no quorum:** we don't need consistency between instances. We need
only mutual exclusion. A lease provides that.

**Failure mode:** *lease expiry → new leader elected* within ~15s.

### 4.3 The subtle dependency

CM and scheduler **don't need quorum** for their own election — but they
**do depend on etcd** for storing the lease.

```
CM's lease is an etcd object.
To renew it, CM must write to etcd.
To write to etcd, etcd needs quorum.
```

So even though CM has no quorum requirement *of its own*, its ability to
function depends on etcd having quorum:

| etcd state | CM can renew lease? | CM failover possible? |
|------------|:-------------------:|:---------------------:|
| 3/3 quorum | ✅ | ✅ |
| 2/3 quorum | ✅ | ✅ |
| 1/3 no quorum | ❌ | ❌ (writes fail) |

### 4.4 The two-layer model

```
           ┌───────────────────────────────────┐
           │   Control-plane logic layer       │
           │                                   │
           │   Scheduler   Controller-manager  │
           │      │            │               │
           │      └──────┬─────┘               │
           │             ▼                     │
           │     Leader Election               │
           │     (single-writer Lease,         │
           │      no quorum)                   │
           └──────────────┬────────────────────┘
                          │
                          │ depends on
                          ▼
           ┌───────────────────────────────────┐
           │   Data / state layer              │
           │                                   │
           │           etcd                    │
           │        (Raft consensus,           │
           │         needs quorum)             │
           └───────────────────────────────────┘
```

| Component | Question it answers |
|-----------|---------------------|
| **CM/scheduler Lease** | "Who is actively doing the work?" |
| **etcd Raft** | "Can the cluster agree on and persist state?" |

**Different problems, different mechanisms.**

### 4.5 Side-by-side comparison

| Aspect | etcd | CM / scheduler |
|--------|------|----------------|
| Mechanism | Raft consensus | Lease (single-writer lock) |
| Requires majority | ✅ Yes | ❌ No |
| 3 members → tolerate | 1 failure | 2 failures |
| 2 of 3 down | ❌ Cluster dead | ✅ Cluster works (if etcd fine) |
| Data replicated | ✅ | ❌ (no replication concept) |
| Failure mode | Loss of quorum → read-only | Holder dies → new leader |
| What it protects | Consistency of state | Single-writer correctness |

### 4.6 The interview-ready one-liner

> **etcd is a consensus problem: it needs majority to make progress.
> Controller-manager and scheduler are mutual-exclusion problems: they
> need only a single-writer lock. They look similar because both use the
> word "leader", but their math and failure modes are completely
> different.**

---

# EXPERIMENTS

---

## 5. Experiment Set A — Understanding Leader Election

### Goal

Observe how CM and scheduler choose their active instance, and confirm
the lease-based mechanism.

### 5.1 Baseline — what a Lease looks like

```bash
kubectl get lease -n kube-system kube-controller-manager -o yaml
```

Example output (trimmed):

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: cp02.faekcorp.lab_af203935-8c81-4232-8871-3620d9844b26
  leaseDurationSeconds: 15
  acquireTime: "2026-10-05T18:04:36.917Z"
  renewTime: "2026-10-05T19:05:28.698Z"
  leaseTransitions: 3
```

Same structure for `kube-scheduler`.

### 5.2 Observed behavior during testing

| Component | Initial holder | Transitions observed |
|-----------|---------------|----------------------|
| controller-manager | cp02 | 2 → 3 → 4 |
| scheduler | cp01 | 2 → 3 |

Each failover incremented `leaseTransitions` by 1 and updated
`holderIdentity` to the new leader node.

### 5.3 What triggers leadership change

Confirmed: leadership moves only when the **container process** that
renews the lease stops renewing. This means:

- `systemctl stop kubelet` → ❌ no effect (container still renewing)
- `kubectl delete pod` (mirror) → ❌ no effect
- `crictl stop <cm-container>` with kubelet running → ❌ container restarts in ~2s
- `crictl stop <cm-container>` with kubelet stopped → ✅ container stays dead, lease expires after 15s, failover occurs

### 5.4 Answer to Question 1

**Leader election does NOT use quorum.** It uses a single-writer Lease.
See Section 4 for the full comparison.

---

## 6. Experiment Set B — Stopping cp01 and cp02 (apiserver included)

### Goal

Determine what happens to the cluster when two of three control-plane
nodes are broken.

### 6.1 What we did

On **cp01** and **cp02**:

1. Stopped kubelet
2. Stopped controller-manager container
3. Stopped scheduler container
4. Stopped apiserver container

We did **NOT** stop etcd.

### 6.2 Node state during the experiment

```
cp01:  kubelet ❌   CM ❌   scheduler ❌   apiserver ❌   etcd ✅
cp02:  kubelet ❌   CM ❌   scheduler ❌   apiserver ❌   etcd ✅
cp03:  kubelet ✅   CM ✅   scheduler ✅   apiserver ✅   etcd ✅
```

### 6.3 What we observed

**Node status:**

```
cp01   NotReady
cp02   NotReady
cp03   Ready
worker1 Ready
worker2 Ready
```

**Leader election moved to cp03:**

```bash
kubectl get leases -n kube-system kube-controller-manager
# kube-controller-manager   cp03.faekcorp.lab_cdf56905-...
kubectl get leases -n kube-system kube-scheduler
# kube-scheduler            cp03.faekcorp.lab_24a2c426-...
```

**HAProxy showed cp01 and cp02 as DOWN:**

```
cp01   DOWN   L4CON in 0ms   14m56s
cp02   DOWN   L4CON in 0ms   13m36s
cp03   UP     L4OK in 0ms     5h32m
```

**Yet deployments still worked:**

```bash
kubectl create deploy nginx-deploy --image=nginx --replicas 6
# deployment.apps/nginx-deploy created
# 6/6 Pods Running, spread across worker1 and worker2
```

### 6.4 Why the cluster survived

The critical component was **etcd**. Still running on all three nodes.

```
etcd members alive: 3 of 3
Quorum:             2 of 3 required
Result:             ✅ intact
```

The full chain was intact because cp03 could play every role:

```
kubectl → HAProxy → cp03 apiserver
                        │
                        ▼
                    etcd (3/3 alive, quorum OK)
                        │
                        ▼
                    cp03 CM (leader)
                        │
                        ▼
                    ReplicaSet created
                        │
                        ▼
                    cp03 scheduler (leader)
                        │
                        ▼
                    Pods assigned to workers
                        │
                        ▼
                    worker kubelets run the Pods
```

### 6.5 The four kinds of availability

The experiment demonstrated that "cluster availability" decomposes into
**four independent kinds**:

| Kind | Question | In our experiment |
|------|----------|-------------------|
| **API availability** | Can I talk to Kubernetes? | ✅ via cp03 |
| **Control-plane reconciliation** | Can K8s react to desired state? | ✅ via cp03's CM + scheduler |
| **Datastore availability** | Can K8s persist state? | ✅ etcd 3/3 |
| **Workload execution** | Can Pods actually run? | ✅ workers unaffected |

**All four were intact despite two of three control-planes being broken.**

### 6.6 The delete-and-recreate test

We deleted the deployment and created a new one with 8 replicas. It
succeeded. That proved:

- Not just "old Pods happened to keep running"
- The **active control path** was genuinely exercised
- CP03 was truly running the control plane

### 6.7 Answer to Question 2

**It depends entirely on what you stop.**

| What's stopped on cp01 + cp02 | etcd alive | Cluster works? |
|-------------------------------|-----------|----------------|
| kubelet only | 3/3 | ✅ |
| kubelet + CM + scheduler | 3/3 | ✅ |
| kubelet + CM + scheduler + apiserver | 3/3 | ✅ |
| kubelet + etcd | 2/3 (cp03 alone) | ❌ quorum lost |
| kubelet + apiserver + etcd | 2/3 | ❌ |
| All (containerd stop) | 1/3 or less | ❌ |

**The determining factor is etcd quorum.** Everything else is secondary.

### 6.8 The crucial insight

> A 3-node control plane can survive losing 2 nodes — **as long as etcd
> quorum is preserved**. Practically, you can lose at most **1 etcd
> member**, regardless of how many other components you lose.

---

## 7. Experiment Set C — Making HAProxy Show a Backend as DOWN

### Goal

Understand what actually triggers HAProxy to mark a backend as DOWN.

### 7.1 What we tried (and what happened)

| Action on a node | HAProxy result |
|------------------|----------------|
| `systemctl stop kubelet` | ✅ Still UP |
| `kubectl delete pod` (mirror) | ✅ Still UP |
| `crictl stop` on CM container | ✅ Still UP |
| `crictl stop` on scheduler container | ✅ Still UP |
| `crictl stop` on apiserver container | ❌ DOWN (after ~10-15s) |
| `systemctl stop containerd` | ❌ DOWN (after ~10-15s) |

**HAProxy flipped to DOWN only when the apiserver process actually died.**

### 7.2 What HAProxy actually checks

```
HAProxy sends a TCP SYN to <backend>:6443
   │
   ├── SYN-ACK received → backend UP
   │
   └── No response / RST → backend DOWN
```

It does **not**:

- Call `/readyz`
- Verify TLS
- Check etcd quorum
- Check kubelet status
- Look at leader-election leases
- Ask Kubernetes anything

**HAProxy's only question:** "Is something listening on port 6443?"

### 7.3 What the dashboard shows

```
cp01   DOWN   L4CON in 0ms   14m56s
cp02   DOWN   L4CON in 0ms   13m36s
cp03   UP     L4OK in 0ms     5h32m
```

- **L4OK** = TCP connect succeeded
- **L4CON** = TCP connect failed (connection-level failure)

### 7.4 Why the delay is ~10-15s

HAProxy's backend check configuration typically:

```
inter 2s      # check every 2 seconds
fall 3        # 3 consecutive failures → DOWN
```

Worst case: 2s × 3 = 6s of the port being closed. Observed ~10-15s
including timing jitter and the point at which we refreshed the dashboard.

### 7.5 What can trigger HAProxy to mark a backend DOWN

| Cause | Effect |
|-------|--------|
| apiserver container stopped | :6443 no longer listening → DOWN |
| containerd stopped | All containers die → DOWN |
| Node powered off | Node unreachable → DOWN |
| Firewall / NACL blocks :6443 | TCP can't connect → DOWN |
| apiserver listening on wrong port | TCP to :6443 fails → DOWN |
| Network partition | TCP times out → DOWN |

**All of them share one property:** TCP connect to `:6443` fails.

### 7.6 Why HAProxy is a "shallow" health check

HAProxy says **UP** even when the node is otherwise deeply unhealthy:

- `kubectl get nodes` → NotReady
- `kubectl get pods` → shows stale Running
- etcd quorum lost
- CM/scheduler down

None of these affect TCP reachability of `:6443`. So HAProxy stays UP.

**HAProxy is a TCP probe, not a Kubernetes-aware health checker.**

### 7.7 Answer to Question 3

**HAProxy only cares about TCP reachability of port 6443.**

To make HAProxy mark a backend DOWN, you must:

1. Kill the apiserver process (or block the port)
2. Wait ~10-15 seconds

Everything else you can break on a node (kubelet, CM, scheduler, mirror
pods) — HAProxy won't notice.

### 7.8 The four health questions

| Component | Question it answers |
|-----------|---------------------|
| **Node Lease** | "Is this kubelet still communicating?" |
| **CM/Scheduler Lease** | "Who is currently the active controller/scheduler?" |
| **etcd quorum** | "Do we still have enough members to reach consensus?" |
| **HAProxy L4 check** | "Can I establish TCP to `<CP>:6443`?" |

They can disagree. That's not a bug.

---

## 8. Experiment Set D — etcd Quorum Loss

### Goal

Directly demonstrate etcd quorum behavior.

### 8.1 Starting state

From Experiment Set B:

```
cp01:  kubelet ❌   CM ❌   scheduler ❌   apiserver ❌   etcd ✅
cp02:  kubelet ❌   CM ❌   scheduler ❌   apiserver ❌   etcd ✅
cp03:  kubelet ✅   CM ✅   scheduler ✅   apiserver ✅   etcd ✅
```

### 8.2 Stopping etcd on cp01 — still works

```bash
ssh ubuntu@cp01.lab
sudo systemctl stop kubelet
sudo crictl ps --name etcd
# note the container ID
sudo crictl stop <etcd-cp01-container-id>
```

Result:

```
etcd members alive: 2 of 3 (cp02, cp03)
Quorum:             2 required
Result:             ✅ intact
```

**Observed:** `kubectl` continued to work. Reads and writes both
succeeded. New deployments could still be created.

**Why:** 2 of 3 is still quorum. The cluster tolerated one etcd member
being gone.

### 8.3 Stopping etcd on cp02 — cluster breaks

```bash
ssh ubuntu@cp02.lab
sudo systemctl stop kubelet
sudo crictl ps --name etcd
# note the container ID
sudo crictl stop <etcd-cp02-container-id>
```

Result:

```
etcd members alive: 1 of 3 (cp03 alone)
Quorum:             2 required
Result:             ❌ LOST
```

**Observed — a cascade of errors.**

**Create fails with timeout:**

```bash
kubectl create deploy nginx-deploy --image=nginx --replicas 4
```

```
error: failed to create deployment: Timeout: request did not complete
within requested timeout - context deadline exceeded
```

**Delete fails with EOF:**

```bash
kubectl delete deploy nginx-deploy
```

```
Unable to connect to the server: unexpected EOF
```

**Then everything returns `Forbidden`:**

```bash
kubectl delete deploy nginx-deploy
```

```
Error from server (Forbidden): deployments.apps "nginx-deploy" is
forbidden: User "kubernetes-admin" cannot delete resource "deployments"
in API group "apps" in the namespace "default"
```

```bash
kubectl get pods
```

```
Error from server (Forbidden): pods is forbidden: User "kubernetes-admin"
cannot list resource "pods" in API group "" in the namespace "default"
```

```bash
kubectl get nodes
```

```
Error from server (Forbidden): nodes is forbidden: User "kubernetes-admin"
cannot list resource "nodes" in API group "" at the cluster scope
```

### 8.4 Why these specific errors

**`context deadline exceeded`:** the request reached the apiserver, the
apiserver tried to write to etcd, etcd couldn't commit, the request
hung, kubectl timed out.

**`unexpected EOF`:** the apiserver gave up on the etcd call and closed
the client connection mid-response. kubectl saw a truncated HTTP stream.

**`Forbidden: User "kubernetes-admin" cannot ...`:** This looks like RBAC
but is not. Here's what's really happening:

```
kubectl sends request
    │
    ▼
apiserver:
  1. Authenticate  →  ✅ (client cert is valid, no etcd needed)
  2. Authorize     →  needs to read RBAC objects from etcd
    │
    ▼
etcd can't serve reliably (no quorum)
    │
    ▼
authorization fails
    │
    ▼
apiserver returns "Forbidden" (fails closed)
```

**The user's permissions are intact.** The problem is that the
authorization check itself requires etcd reads that can't complete.

`Forbidden` in this context means "I couldn't verify your permissions,
so I'm denying by default."

### 8.5 The exact quorum boundary

| Members alive | Result |
|---------------|--------|
| 3 of 3 | ✅ Full function |
| **2 of 3** | **✅ Full function** |
| **1 of 3** | **❌ Cluster effectively dead** |

The transition happened **exactly** between "cp01 alone stopped" and
"cp01 and cp02 stopped." That's the quorum rule in action.

### 8.6 Why we initially suspected kubeconfig

The `Forbidden` errors looked like authorization failures. We considered:
*Is the kubeconfig somehow tied to a specific node?*

**No — and we proved it by reasoning:**

- kubeconfig uses a client certificate signed by the cluster CA
- The certificate does not belong to any specific node
- The endpoint is `haproxy.<domain>:6443` (the load balancer)
- **If kubeconfig were the problem, failure would have started when
  cp01's etcd died — not when cp02's did.**

The failure correlated **exactly** with the quorum boundary, so it was
quorum, not kubeconfig.

### 8.7 Existing workloads kept running

Throughout the quorum loss, Pods already running on worker1 and worker2
continued to run. Container processes don't need the Kubernetes API to
keep executing.

Only **changes** — creating, deleting, updating objects — required the
API path, and that path was blocked by lost quorum.

### 8.8 Answer to Question 4

**etcd quorum is the hard floor of Kubernetes HA.** No amount of
redundancy in apiserver, CM, scheduler, or kubelet can compensate for
lost etcd quorum. Everything else depends on it.

---

## 9. Experiment Set E — Apiserver Crash-Loop from Quorum Loss

### Goal

Understand what happens to an apiserver when etcd loses quorum — and
why HAProxy then marks the node DOWN even without us stopping anything.

### 9.1 The surprise

After Experiment Set D (cp01 + cp02 etcd stopped, cp03 alone alive), we
noticed:

**HAProxy dashboard showed all three CPs DOWN.**

```
cp01   DOWN   L4CON   51m14s
cp02   DOWN   L4CON   49m54s
cp03   DOWN   L4CON   15m6s    ← cp03 also DOWN, we didn't touch it
```

**We never stopped cp03's apiserver.** Yet it went DOWN.

### 9.2 What we found on cp03

```bash
sudo crictl ps -a | grep -i api
```

```
99b2452df9056   09e313e26e711   3 minutes ago   Exited   kube-apiserver   10   kube-apiserver-cp03.faekcorp.lab
```

Two key facts:

- **STATE: Exited** — apiserver container is not running
- **ATTEMPT: 10** — this container has been restarted 10 times

Meanwhile, on cp03:

```bash
sudo crictl ps | grep -E 'etcd|scheduler|controller'
```

```
b02efebb0073b   ...   Running   kube-scheduler             (26 minutes ago)
953b1b77fa44e   ...   Running   kube-controller-manager    (26 minutes ago)
2fcb483b4f7a1   ...   Running   etcd                       (6 hours ago)
```

**So on cp03:**

- etcd ✅ running
- scheduler ✅ running
- controller-manager ✅ running
- apiserver ❌ Exited, in crash-loop (attempt 10)

### 9.3 kubelet status

```bash
sudo systemctl status kubelet.service
```

```
Active: active (running) since Mon 2026-10-05 15:09:50 UTC; 6h ago
```

kubelet is running. It's reconciling. It's trying to run the apiserver
from the manifest. But the apiserver keeps exiting.

### 9.4 The kubelet logs — decode them

```bash
journalctl -u kubelet.service -n 20 --no-pager
```

Repeated errors like:

```
kubelet_node_status.go:474  "Error updating node status, will retry"
  err="error getting node \"cp03.faekcorp.lab\":
  Get \"https://10.70.41.250:6443/api/v1/nodes/cp03.faekcorp.lab\":
  dial tcp 10.70.41.250:6443: connect: connection refused"

kubelet_node_status.go:461  "Unable to update node status"
  err="update node status exceeds retry count"

controller.go:201  "Failed to ensure lease exists, will retry"
  err="...namespaces/kube-node-lease/leases/cp03.faekcorp.lab:
  dial tcp 10.70.41.250:6443: connect: connection refused"

status_manager.go:1045  "Failed to get status for pod"
  err="...namespaces/kube-system/pods/etcd-cp03.faekcorp.lab:
  dial tcp 10.70.41.250:6443: connect: connection refused"
```

**Every error points to the same address: `10.70.41.250:6443`** — cp03's
own apiserver port. **Nothing is listening there.**

### 9.5 Why "connection refused" matters

| Error | Meaning |
|-------|---------|
| `i/o timeout` | Packets silently dropped (firewall) |
| **`connection refused`** | **SYN reached host, but nothing is listening on that port. Kernel sent RST.** |
| `no route to host` | No route to destination |

`connection refused` proves:

1. Network path works (SYN reached cp03)
2. Host is up (kernel responded with RST)
3. **No process is bound to port 6443** — the apiserver isn't running

### 9.6 Why did the apiserver disappear?

We didn't stop it. Here's the causal chain:

```
T0:  We stopped etcd on cp01
       │
T1:  We stopped etcd on cp02
       │
T2:  etcd cluster: 1/3 alive → NO QUORUM
       │
T3:  cp03's apiserver tries to start (kubelet reconciles from manifest)
       │
     apiserver connects to etcd endpoints:
        cp01:2379 → ❌
        cp02:2379 → ❌
        cp03:2379 → ✅ alive but "no leader" (no quorum)
       │
T4:  apiserver's startup health check fails
       │
T5:  apiserver exits with an error
       │
T6:  kubelet sees exit → restarts → same failure → loop
       │
T7:  Attempt count increments: 1, 2, 3, ... 10
       │
T8:  HAProxy's TCP check to :6443 fails → cp03 marked DOWN
       │
T9:  All three apiservers DOWN → HAProxy has no backend → kubectl fails
```

**The apiserver crash-looped because etcd had no quorum.**

### 9.7 The lesson — apiserver depends on etcd quorum

The apiserver, at startup, performs a health check against etcd. If it
can't read and write consistently, it refuses to start.

**This means:** etcd quorum isn't just "needed for writes." It's needed
for the apiserver to even **exist as a running process**.

### 9.8 Why kubelet couldn't fix it

kubelet's job is: "Run what the manifest says."

kubelet can't fix a broken apiserver because:

1. The manifest is fine — it says "run the apiserver"
2. kubelet runs the apiserver — but the apiserver exits
3. kubelet runs it again — same exit
4. kubelet can't diagnose why; it just runs containers and watches them

**kubelet has no etcd-quorum awareness.** It just retries.

### 9.9 The container ID changes with every restart

When we tried:

```bash
sudo crictl logs 99b2452df9056
```

We got:

```
container "99b2452df9056": not found
```

Because between the `crictl ps -a` and the `crictl logs`, kubelet had
already cleaned up and started a new attempt. **Each restart gets a new
container ID.**

To catch the apiserver's actual error:

```bash
# Watch for new attempts
watch -n1 'sudo crictl ps -a | grep kube-apiserver'

# Or look at kubelet's own pod events
sudo journalctl -u kubelet --since "5 minutes ago" | grep -A5 kube-apiserver
```

### 9.10 Answer to Question 5

**The apiserver disappeared because etcd lost quorum.** The fix is
singular and simple: **restore etcd quorum.**

### 9.11 The critical chain to remember

```
etcd quorum is the foundation of everything else.
       │
       ▼
apiserver can't run without it.
       │
       ▼
Without apiserver, no API, no coordination, no writes.
       │
       ▼
HAProxy, kubelet, CM, scheduler all fail downstream.
```

**Never troubleshoot the downstream symptoms first.** Always ask:
*is etcd quorate?* If no, nothing else matters until quorum is restored.

### 9.12 Recovery — Restoring Quorum Restores the Apiserver Automatically

Once we identified the root cause (etcd quorum loss), we took the
smallest possible corrective action: **we started kubelet on one node**
(cp01). That was the only change.

```bash
ssh ubuntu@cp01.lab
sudo systemctl start kubelet
```

We did **not**:

- Restart the apiserver manually on cp03
- Touch cp03 in any way
- Edit any manifest
- Restart containerd anywhere
- Restart HAProxy

**And yet, within seconds:**

- cp03's apiserver stopped crash-looping and began running
- HAProxy showed **cp01 UP** and **cp03 UP**
- kubectl started working again
- kubelet's API errors on cp03 stopped

**The recovery happened entirely on its own once etcd quorum was
restored.**

#### Why cp03's apiserver recovered without intervention

Trace the recovery chain:

```
t=0:   You start kubelet on cp01
         │
t=+1s: kubelet starts etcd-cp01 container from the manifest
         │
t=+2s: etcd-cp01 rejoins the Raft cluster
         │  (cp02's etcd dead, cp03's etcd alive)
         ▼
t=+3s: etcd quorum restored (2 of 3: cp01 + cp03)
         │  Raft leader elected
         │  etcd accepts writes again
         ▼
t=+3s: cp03's etcd can now serve reads/writes properly
         │  (It had been running all along; without quorum
         │   it couldn't make progress.)
         ▼
t=+5s: cp03's apiserver, on its next restart attempt,
       connects to etcd — health check passes
         │
         │  apiserver stays running
         │  apiserver binds to :6443
         ▼
t=+6s: apiserver is listening on 10.70.41.250:6443
         │  HAProxy's next health check succeeds
         ▼
t=+7s: HAProxy marks cp03 UP (L4OK)
         │  cp01's apiserver also came up (started by
         │   kubelet from manifest, connected to healthy etcd)
         ▼
t=+8s: HAProxy marks cp01 UP
         │
t=+10s: kubectl works again
```

**Every downstream symptom resolved itself.** The only action required
was fixing the root cause.

#### Why cp01 came back UP too

Starting kubelet on cp01 caused kubelet to read
`/etc/kubernetes/manifests/` and start **all** control-plane containers
on cp01 — not just etcd:

- etcd-cp01 ✅
- kube-apiserver-cp01 ✅
- kube-controller-manager-cp01 ✅
- kube-scheduler-cp01 ✅

The apiserver on cp01 started successfully because it could now reach a
quorate etcd. So starting kubelet on cp01 both **restored quorum** and
**restored one apiserver** in a single action.

#### Why cp02 stayed DOWN

We only started kubelet on cp01. cp02's kubelet was still stopped, so
cp02's containers were still gone. HAProxy continued to show cp02 DOWN
until we started its kubelet too.

#### What this proves

| Observation | What it confirms |
|-------------|------------------|
| We did not touch cp03 | The apiserver's recovery was not caused by anything on cp03 |
| We did not restart the apiserver container | It recovered because etcd became healthy |
| The only action was `systemctl start kubelet` on cp01 | The **only** change was etcd quorum restoration |
| cp03's apiserver came back automatically | It had been waiting on etcd quorum the whole time |
| HAProxy flipped cp03 UP with no intervention | HAProxy just saw TCP connect succeed — as expected |
| kubelet was running on cp03 the whole time | kubelet kept retrying the apiserver; it needed etcd healthy to succeed |

**The recovery is the definitive proof of the diagnosis.** No other
explanation accounts for the observation that restoring quorum on one
node caused a crash-looping component on a *different* node to recover
without being touched.

#### The retry-forever principle

Kubernetes components do not "give up." They reconcile forever.

- kubelet kept trying to run the apiserver, no matter how many times it
  exited
- The apiserver kept failing, no matter how many times it was started
- **Nobody gave up.** The system just retried.
- When the underlying condition changed (etcd got quorum), the retries
  started succeeding — **automatically**

This is **convergent reconciliation**: the system does not need to be
"told" that the environment has healed. It just keeps trying until
success. Success arrives naturally when the prerequisite condition is
restored.

**This is why Kubernetes HA is robust:** there is no "give up" state.
There is only "retry forever." Recovery is guaranteed once the root
cause is fixed.

#### The correct recovery order

The natural instinct when debugging is to fix the visible symptoms:

- ❌ Restart the apiserver manually
- ❌ Restart HAProxy
- ❌ Restart kubelet on cp03

Those actions would **not** have fixed the problem. The apiserver would
have crash-looped again immediately.

**The correct action is always: fix the root cause (etcd quorum). Let
convergence do the rest.**

```
Correct recovery order:
   1. Restore etcd quorum   ← fix the root cause
   2. Wait                   ← let convergence happen
   3. Verify                 ← check HAProxy, kubectl, nodes
```

---

# REFERENCE

---

## 10. Findings Reference Tables

### 10.1 Action vs Outcome

| Action | etcd quorum intact? | Cluster works? | HAProxy UP? |
|--------|:-------------------:|:--------------:|:-----------:|
| Stop kubelet on 1 node | ✅ | ✅ | ✅ |
| Stop kubelet on 2 nodes | ✅ | ✅ | ✅ |
| Stop CM + scheduler on 2 nodes | ✅ | ✅ | ✅ |
| Stop apiserver on 2 nodes | ✅ | ✅ | ❌ for those nodes |
| Stop etcd on 1 node | ✅ (still quorum) | ✅ | ✅ (if apiserver alive) |
| Stop etcd on 2 nodes | ❌ | ❌ | ✅ then ❌ |
| Stop everything (containerd) on 2 nodes | ❌ | ❌ | ❌ |

### 10.2 Component Failure Tolerance

| Component | Tolerates | Limit |
|-----------|-----------|-------|
| etcd | 1 of 3 down | **Only 1** — quorum = 2 of 3 |
| kube-apiserver | 2 of 3 down | As long as 1 is reachable |
| kube-controller-manager | 2 of 3 down | As long as 1 can hold the lease |
| kube-scheduler | 2 of 3 down | As long as 1 can hold the lease |
| kubelet (per node) | Node becomes NotReady | Containers keep running |
| HAProxy backend | N/A | Only fails if port 6443 is closed |

### 10.3 Error Signature vs Cause

| Error | Underlying cause |
|-------|-----------------|
| `context deadline exceeded` | etcd can't commit (no quorum) |
| `unexpected EOF` | apiserver gave up on etcd, closed connection |
| `Forbidden` (unexpected) | authorization check failed to read from etcd |
| `Unable to connect to the server` | all apiservers unreachable |
| `dial tcp ...:6443: connect: connection refused` | apiserver not listening on that port |
| `L4CON` in HAProxy | TCP connect failed |
| `L4OK` in HAProxy | TCP connect succeeded (nothing more) |

### 10.4 The Four Independent HA Layers

| Layer | Mechanism | Timeout | Trigger |
|-------|-----------|---------|---------|
| **etcd quorum** | Raft consensus | ~1s | Member majority lost |
| **Node Lease** | Lease renewal by kubelet | ~40s | kubelet stopped |
| **Leader-election Lease** | Lease renewal by CM/scheduler | ~15s | Container process stopped |
| **HAProxy** | TCP connect to :6443 | ~10-15s | apiserver process stopped |

### 10.5 Recovery Requirements by Component

| Broken component | To recover, you need |
|------------------|---------------------|
| kubelet | `systemctl start kubelet` |
| CM container | `systemctl start kubelet` (kubelet recreates it) |
| scheduler container | Same |
| **apiserver container** | **kubelet running + etcd quorum restored** |
| **etcd** | `systemctl start kubelet` (kubelet recreates it) |
| **whole cluster (all apiservers down)** | **Restore etcd quorum first** |

### 10.6 Question vs Answer Confirmed by Recovery

| Question we had | Confirmed by recovery |
|-----------------|----------------------|
| Did the apiserver crash because of something we did to it directly? | **No.** We didn't touch cp03. Restoring quorum elsewhere fixed it. |
| Is etcd quorum really the floor of HA? | **Yes.** Restoring quorum on cp01 healed a crash-looping apiserver on cp03. |
| Will the cluster recover automatically? | **Yes.** No manual intervention on cp03 was needed. |
| Does kubelet give up on the apiserver? | **No.** It retries forever until success. |
| Does HAProxy need to be told a backend is back? | **No.** It just sees TCP connect succeed on the next check. |

---

## 11. The Complete HA Mental Model

```
                    ┌──────────────────────────────┐
                    │        etcd (3 members)      │
                    │                              │
                    │  ⚠️  Only this layer needs    │
                    │      quorum (majority).      │
                    │                              │
                    │  3/3 alive  → ✅ works        │
                    │  2/3 alive  → ✅ works        │
                    │  1/3 alive  → ❌ dead         │
                    │                              │
                    │  No quorum → apiserver        │
                    │  can't even start             │
                    └──────────────┬───────────────┘
                                   │
                                   │  provides state storage
                                   │  + startup dependency
                                   ▼
                    ┌──────────────────────────────┐
                    │   kube-apiserver (3 nodes)   │
                    │                              │
                    │  Active/active.              │
                    │  Any instance can serve.     │
                    │  Writes need etcd quorum.    │
                    │  Authz checks need etcd.     │
                    │  Startup needs etcd quorum.  │
                    └──────────────┬───────────────┘
                                   │
                                   │  exposes API
                                   ▼
                    ┌──────────────────────────────┐
                    │   HAProxy (external LB)      │
                    │                              │
                    │  TCP connect to :6443 only.  │
                    │  Doesn't know about etcd,    │
                    │  kubelet, or leadership.     │
                    │  Only "is port 6443 open?"   │
                    └──────────────┬───────────────┘
                                   │
                                   │  routes clients
                                   ▼
                    ┌──────────────────────────────┐
                    │   controller-manager /       │
                    │   scheduler (3 instances)    │
                    │                              │
                    │  Leader election via Lease.  │
                    │  No quorum needed.           │
                    │  Needs etcd writable to      │
                    │  acquire / renew lease.      │
                    └──────────────┬───────────────┘
                                   │
                                   │  instruct nodes
                                   ▼
                    ┌──────────────────────────────┐
                    │      kubelet (per node)      │
                    │                              │
                    │  Runs containers.            │
                    │  Reports via Node Lease.     │
                    │  Independent of control-     │
                    │  plane components on node.   │
                    └──────────────────────────────┘
```

### 11.1 Three golden rules

1. **etcd quorum is the floor.** No matter how healthy everything else
   is, if etcd loses quorum, the cluster stops accepting writes — and
   the apiserver may crash-loop entirely.
2. **HAProxy is a TCP probe.** Its "UP" tells you the apiserver port is
   open. Nothing more.
3. **Recovery order matters.** Restore etcd first. Everything else
   follows automatically.

### 11.2 The recovery principle

> **Kubernetes components do not "give up." They reconcile forever.**
>
> When the root cause is fixed (etcd quorum restored), every downstream
> component recovers automatically — the apiserver, the kubelet's API
> operations, HAProxy's backend status.
>
> **Fix the root cause; let convergence do the rest.**
>
> This was directly observed: starting kubelet on **one** node restored
> etcd quorum, which caused a crash-looping apiserver on a **different**
> node to recover on its own, which caused HAProxy to flip that backend
> back to UP — all without any manual intervention on the recovered node.

### 11.3 The conceptual core

- **etcd → quorum-based** (Raft consensus, majority required)
- **CM/scheduler → lease-based** (single-writer lock, no quorum)
- **HAProxy → TCP-based** (port reachability only)
- **Node Lease → kubelet heartbeat** (node health reporting)

**Different mechanisms for different problems.**

---

## 12. Recovery Procedures Used

### 12.1 Recovering from kubelet stopped on a node

```bash
ssh ubuntu@<node>
sudo systemctl start kubelet
sudo systemctl status kubelet --no-pager
```

Node returns to `Ready` in ~40s. Containers were never affected.

### 12.2 Recovering from CM/scheduler container stopped

```bash
ssh ubuntu@<node>
sudo systemctl start kubelet
# kubelet recreates the container from its manifest
sudo crictl ps | grep -E 'controller-manager|scheduler'
```

### 12.3 Recovering from apiserver stopped

**Two cases:**

**Case 1 — etcd quorum is intact:**

```bash
ssh ubuntu@<node>
sudo systemctl start kubelet
sudo crictl ps | grep kube-apiserver
```

Apiserver starts normally. HAProxy's next health check succeeds.

**Case 2 — etcd quorum is lost:**

You **must** restore etcd quorum **first**. See 12.4. The apiserver will
recover automatically once etcd has quorum.

### 12.4 Recovering from etcd quorum loss

Bring back etcd on **any one** of the stopped nodes:

```bash
ssh ubuntu@<node>
sudo systemctl start kubelet
sudo crictl ps | grep etcd
```

Within ~10s, etcd rejoins and quorum is restored. Writes resume
immediately. Any apiserver crash-looping will recover on its next
restart attempt.

### 12.5 Full restoration checklist

```bash
# On each previously-affected node:
sudo systemctl start kubelet

# Wait ~60s, then verify:
kubectl get nodes
# All Ready

kubectl get pods -n kube-system -o wide
# All system pods Running

kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml
# Both have active holders with fresh renewTime

curl -k https://haproxy.<domain>:6443/readyz
# ok
```

**Note:** the leases do **not** move back to the original nodes.
Leadership stays where it moved. `leaseTransitions` does not decrement.

### 12.6 Recovering from a crash-looping apiserver

If the apiserver is crash-looping, the recovery is the same as etcd
recovery — fix the underlying etcd problem and the apiserver will
recover on its next restart attempt.

**Do not** attempt to manually restart the apiserver. That will not
help. Fix the root cause instead.

To observe the recovery:

```bash
# Watch apiserver attempts stop
watch -n1 'sudo crictl ps -a | grep kube-apiserver'

# Watch kubelet errors stop
sudo journalctl -u kubelet -f
# Once apiserver is reachable, errors will cease

# On bastion
curl -k https://haproxy.<domain>:6443/readyz
```

### 12.7 The universal recovery order

For any etcd-quorum-related failure, follow this order:

```
1. Identify the root cause.
   Is etcd quorate?
   - Check etcd members alive: run crictl ps | grep etcd on each CP
   - If fewer than 2 of 3 alive → quorum lost

2. Fix the root cause.
   Restore etcd on one or more nodes:
   - Start kubelet on the node (kubelet recreates etcd from manifest)

3. Wait for convergence.
   Do NOT manually restart downstream components.
   - The apiserver will recover on its next retry
   - HAProxy will flip the backend UP on its next check
   - kubelet's API errors will stop

4. Verify recovery.
   - kubectl get nodes → all Ready
   - HAProxy dashboard → all backends UP
   - curl -k https://haproxy.<domain>:6443/readyz → ok
```

**The most important step is #3.** The instinct is to fix the visible
symptoms (restart the apiserver, restart HAProxy). Resist it. Kubernetes
converges on its own. **Fix the root cause and step back.**

---

## Summary — What These Experiments Proved

| # | Finding | Proven by |
|---|---------|-----------|
| 1 | Leader election is lease-based, not quorum-based | Configuring and observing CM/scheduler leases |
| 2 | CM/scheduler depend on etcd for their leases | No failover when etcd has no quorum |
| 3 | The cluster survives losing 2 of 3 control planes' components | Stopping kubelet, CM, scheduler, apiserver on cp01/cp02 |
| 4 | **But** it does not survive losing 2 of 3 etcd members | Stopping etcd on cp01 then cp02 |
| 5 | HAProxy only cares about TCP reachability of :6443 | Never reacting to kubelet / CM / scheduler failures |
| 6 | HAProxy marks a backend DOWN only when apiserver is gone | Killing apiserver container → dashboard flipped |
| 7 | etcd quorum loss → `Forbidden` errors that look like RBAC | Observed after stopping 2 of 3 etcd members |
| 8 | kubeconfig is not the weak point — it's etcd quorum | Failure aligned exactly with quorum boundary |
| 9 | Existing workloads keep running through quorum loss | Pods on workers stayed Running during the entire D-test |
| 10 | Leadership does not return to recovered nodes | Leases stayed on their new holders after recovery |
| 11 | etcd uses quorum; CM/scheduler use a single-writer lease | Both observed independently in the same cluster |
| 12 | Apiserver crash-loops when etcd has no quorum | cp03's apiserver attempted 10 times and kept exiting |
| 13 | kubelet keeps trying to run the apiserver even when it can't succeed | kubelet active, reconciling, but errors on all API operations |
| 14 | `connection refused` on :6443 means no process is listening | Diagnosed cp03's apiserver crash-loop |
| 15 | Recovery order matters — etcd first | cp03's apiserver recovered automatically when quorum returned |
| 16 | Kubernetes components retry forever; they never "give up" | kubelet kept restarting the apiserver until etcd was healthy again |
| 17 | Automatic recovery does not require touching the recovered component | Restoring quorum on cp01 healed a crash-looping apiserver on cp03 |

---
