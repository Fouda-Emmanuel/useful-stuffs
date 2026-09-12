# Kubernetes Cluster Upgrade Guide — `kubeadm` (v1.32 → v1.33)

Companion to `kubernetes-cluster-setup.md`. Written to be followed by
someone who has never done a Kubernetes upgrade before, and to double as
your own reference the next time you do this yourself.

- **From:** v1.32.13
- **To:** v1.33.12 (N-1 patch policy — see [§5](#5-the-n-1-patch-policy))
- **Topology:** 1 control plane, 3 workers, 1 bastion, 3 AZs
- **Architecture:** `kubectl` on the bastion only
- **Runtime/CNI:** containerd 2.2.1, Calico v3.30.2 (VXLANCrossSubnet)

> **Before you begin:** do not skip [Phase 0](#9-phase-0--pre-flight-do-not-skip).
> The etcd snapshot it produces is your only rollback if anything below goes
> wrong. Read [Section 10, "If Something Goes Wrong"](#10-if-something-goes-wrong--rollback--disaster-recovery)
> once, before you touch anything, so you're not learning it for the first
> time during an actual incident.

**How to use this doc:** each step is one block of required commands to
copy and paste as-is — run it, read **Why** underneath, then **Verify**
before moving on. Blocks marked **OPTIONAL** are separate on purpose; skip
them freely. Every block says exactly which machine to run it on.

---

## Table of Contents

1. [Versioning, Cadence, and Support](#1-versioning-cadence-and-support)
2. [Version Skew Rules](#2-version-skew-rules)
3. [Why Upgrades Matter](#3-why-upgrades-matter)
4. [Upgrade Strategies](#4-upgrade-strategies)
5. [The N-1 Patch Policy](#5-the-n-1-patch-policy)
6. [CRI and CNI Are Separate Upgrade Tracks](#6-cri-and-cni-are-separate-upgrade-tracks)
7. [PodDisruptionBudgets, Briefly](#7-poddisruptionbudgets-briefly)
8. [How This Differs From a Generic Upgrade Doc](#8-how-this-differs-from-a-generic-upgrade-doc)
9. [Phase 0 — Pre-flight (Do Not Skip)](#9-phase-0--pre-flight-do-not-skip)
10. [If Something Goes Wrong — Rollback & Disaster Recovery](#10-if-something-goes-wrong--rollback--disaster-recovery)
11. [Phase 1 — Control Plane Upgrade](#11-phase-1--control-plane-upgrade)
12. [Phase 2 — Worker Upgrades (one at a time)](#12-phase-2--worker-upgrades-one-at-a-time)
13. [Phase 3 — Bastion kubectl Upgrade + Lockdown](#13-phase-3--bastion-kubectl-upgrade--lockdown)
14. [Phase 4 — Post-Upgrade Validation](#14-phase-4--post-upgrade-validation)
15. [Phase 5 — Reboot Cycle](#15-phase-5--reboot-cycle)
16. [Phase 6 — Cleanup](#16-phase-6--cleanup)
17. [Troubleshooting](#17-troubleshooting)
18. [Lessons Learned From This Upgrade](#18-lessons-learned-from-this-upgrade)
19. [Appendix: Ports, Paths, Reference Commands](#19-appendix-ports-paths-reference-commands)
20. [References](#20-references)

---

## 1. Versioning, Cadence, and Support

### Version format

```
1.32.13
│ │  └── PATCH  → bug/security fixes, no new features
│ └───── MINOR  → new features, API changes, deprecations
└─────── MAJOR  → breaking changes (never happened in Kubernetes)
```
Kubernetes has never bumped the major version — a "new version" almost
always means a new **minor**. Patch upgrades are routine; minor upgrades
need planning, which is the entire reason this runbook exists.

### Release cadence

- A new **minor** ships roughly every 4 months (~3 per year).
- **Patches** land every 1–2 weeks per supported branch, especially right
  after a new minor drops.

### Support window — N, N-1, N-2

At any moment, upstream Kubernetes supports exactly three minors:

```
N      ← current (e.g. 1.33)  — patched
N-1    ← previous (1.32)      — patched
N-2    ← older (1.31)         — patched
N-3    ← (1.30)               — NOT patched, no security fixes
```
Each minor stays supported for about 12 months. Fall outside this window
and you stop getting security fixes — the main reason upgrades aren't
optional. Managed Kubernetes (EKS/AKS/GKE) buys extra months on top of
this; a self-managed cluster like this one is on the strict upstream
schedule.

**You cannot skip minor versions.** `1.31 → 1.33` directly is unsupported —
the path is always `1.31 → 1.32 → 1.33`, one minor at a time. Each minor's
upgrade assumes the previous minor's state (etcd schema and API migrations
are staged this way). Falling multiple minors behind means multiple
separate upgrade windows to catch back up.

---

## 2. Version Skew Rules

Components are allowed to sit at different versions **within limits**
during a rolling upgrade. Breaking these limits causes silent failures, not
loud errors.

| Component | Allowed relative to the API server | Must never be |
|---|---|---|
| kube-controller-manager, kube-scheduler | same minor, or 1 minor older | newer than the API server |
| kubelet (on nodes) | same minor, or up to 2 minors older | newer than the API server |
| kubectl (client) | within ±1 minor of the API server | more than 1 minor off |

**The one rule that matters most: never run a component newer than the API
server.** That's why the order is always fixed — API server (control
plane) first, then workers, one node at a time. During the roll, having the
control plane on the new minor while workers are still on the old one is
normal and supported; it's the mirror image (a worker ahead of the control
plane) that's never allowed.

---

## 3. Why Upgrades Matter

1. **Security** — CVEs are patched only on supported branches. Fall off the
   `N/N-1/N-2` window and a critical CVE gets you nothing.
2. **Compliance** — SOC 2, PCI, HIPAA and similar frameworks generally
   require supported, patched software.
3. **Ecosystem compatibility — the real trap.** Your CNI (Calico), CSI
   drivers, and other addons target *currently supported* Kubernetes
   minors. Stay too far behind and you get cornered: a newer Calico
   requires APIs your old Kubernetes doesn't have, so you can't upgrade
   Calico — but upgrading Kubernetes now means several minor jumps at once,
   which is itself risky. Regular upgrades are how you never get cornered.
4. **New features** — the least urgent reason, but the one people notice.

---

## 4. Upgrade Strategies

| Strategy | What it is | When |
|---|---|---|
| All at once | Upgrade everything simultaneously | dev/test only — full outage window |
| **Rolling** (used here) | One node at a time; ≥N-1 nodes always serving | Production, always |
| Blue/Green | Stand up a second cluster, validate, cut over, retire the old one | High-risk changes (CNI swap, major OS jump) |

Rolling, applied here:
```
1. Upgrade control plane
2. Cordon + drain worker1 → upgrade worker1 → uncordon worker1
3. Cordon + drain worker2 → upgrade worker2 → uncordon worker2
4. Cordon + drain worker3 → upgrade worker3 → uncordon worker3
```
At every step, the nodes not currently being upgraded keep serving traffic.

---

## 5. The N-1 Patch Policy

Within a target minor, don't install the absolute latest patch — install
the one just behind it.

**Why:** by the time you install `N-1`, other teams running the very latest
patch have already surfaced any showstopper bugs. You get almost all the
benefit of the new minor with a little more soak time behind you. This
isn't about being slow; it's about not being first.

**Example:** if `apt-cache madison kubeadm` shows `1.33.13` as the newest
available, target `1.33.12` now, and consider `1.33.13` in a later,
separate maintenance window once it's had time to prove itself.

**Apt version strings have a Debian revision suffix** — always use the full
string:
```
kubeadm=1.33.12-1.1
            │  │
            │  └── Debian package revision
            └───── Kubernetes version
```
`apt-get install kubeadm=1.33.12` (without `-1.1`) will fail — always copy
the exact string from `apt-cache madison`.

---

## 6. CRI and CNI Are Separate Upgrade Tracks

`kubeadm` does **not** upgrade containerd (CRI) or Calico (CNI) — they have
their own, independent upgrade lifecycles. Before every Kubernetes upgrade,
check both against your target version:

```
Target Kubernetes version (e.g. 1.33)
        │
        ▼
Does current Calico support it?  ──NO──► upgrade Calico FIRST
        │ YES
        ▼
Does current containerd support it?  ──NO──► upgrade containerd FIRST
        │ YES
        ▼
Proceed with the Kubernetes upgrade
```

In this build: containerd 2.2.1 and Calico 3.30.2 (which supports
1.31–1.33) both already support the 1.33 target, so **no CRI/CNI upgrade
was needed** — the ideal case, and worth confirming explicitly every time
rather than assuming it.

What `kubeadm upgrade apply` *does* handle automatically: `kube-apiserver`,
`kube-controller-manager`, `kube-scheduler`, `etcd` (via the static pod
manifests), plus rolling `CoreDNS` and `kube-proxy`. `kubelet`, `kubeadm`,
containerd, and Calico are all upgraded manually, on their own schedules.

---

## 7. PodDisruptionBudgets, Briefly

A PDB protects a workload from **voluntary** disruptions — like
`kubectl drain` — but not involuntary ones (a node crashing, an OOM kill).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 3      # or maxUnavailable: 1, or a percentage
  selector:
    matchLabels:
      app: web
```

`kubectl drain` evicts pods through the Eviction API, which checks every
eviction against any matching PDB before allowing it; `kubectl delete pod`
bypasses PDBs entirely. A PDB only counts pods that are actually **Ready**
— a replacement pod that's merely scheduled, but still failing its
readiness probe, doesn't restore the budget, so a drain can appear to
"stall" while it's actually just correctly waiting.

This lab cluster has no PDBs configured — safe here because there's enough
spare replica capacity across the other workers to absorb a drain. In a
real production workload, define a PDB (and a readiness probe) before you
ever drain a node running it.

PDBs are unrelated to Pod Priority: priority affects scheduling and
preemption; only PDBs (plus cordon/drain) protect against planned
maintenance disruptions.

---

## 8. How This Differs From a Generic Upgrade Doc

| Generic doc assumes | This cluster actually has |
|---|---|
| `kubectl` on the control plane | `kubectl` only on the **bastion** — every `kubectl ...` command below runs there |
| containerd 1.7.x | containerd 2.2.1 — fine for 1.33, don't be thrown by version numbers in generic docs |
| HA control plane (`kubeadm upgrade node` on extra CPs) | single control plane — only one `kubeadm upgrade apply`, no HA branch |
| A fixed example target version | whatever `apt-cache madison kubeadm` shows once pointed at v1.33, filtered down one patch per the N-1 policy above |

Command-placement rule for this whole document:
- `kubectl ...` → **bastion**
- `kubeadm ...`, `apt-mark`, `apt-get install`, `systemctl` → the **specific node** being upgraded

---

## 9. Phase 0 — Pre-flight (Do Not Skip)

`kubeadm` does none of this for you — it assumes a healthy cluster and a
prepared operator. It doesn't check cluster health, doesn't snapshot etcd,
and doesn't back up `/etc/kubernetes`. If the upgrade goes wrong, kubeadm's
answer is "that's your problem." All four checks below are mandatory.

### 9.1 Verify cluster health — [BASTION]

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl cluster-info
kubectl get --raw='/readyz?verbose' | tail -20
kubectl -n kube-system exec etcd-controlplane -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint health
```

**Why:** if the cluster is already unhealthy, upgrading won't fix it — it
just makes the failure harder to diagnose. Never upgrade a sick cluster.

**Verify — all of these must be true:** all nodes `Ready`; every pod
`Running` or `Completed`; the `field-selector` command returns nothing;
`/readyz` ends in `readyz check passed`; etcd reports `is healthy`.

### 9.2 Take an etcd snapshot — [CONTROL PLANE]

```bash
# etcdctl isn't installed by default — install the client package first
sudo apt-get update
sudo apt-get install -y etcd-client
etcdctl version

sudo mkdir -p /var/backups/etcd

sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd/pre-upgrade-$(date +%Y%m%d-%H%M%S).db
```

**Why:** this snapshot is your **only** rollback path for the cluster's
data if the upgrade goes catastrophically wrong. It takes under a minute —
there's no good reason to skip it, and for a single-control-plane cluster
like this one, it's the difference between a bad afternoon and rebuilding
from scratch.

**Verify:**
```bash
ls -lh /var/backups/etcd/

SNAP=$(ls -t /var/backups/etcd/*.db | head -1)
sudo ETCDCTL_API=3 etcdctl snapshot status "$SNAP" --write-out=table
```
Expect a table with `HASH`, `REVISION`, `TOTAL KEYS`, `TOTAL SIZE` filled in.

### OPTIONAL — copy the snapshot off the control plane

Do this so the backup survives even if the CP's disk dies. Skip it for a
throwaway lab.

```bash
# On the control plane — make it readable
sudo chown ubuntu:ubuntu /var/backups/etcd/*.db
```
```bash
# On the bastion — pull it over
scp ubuntu@<CP_PRIVATE_IP>:/var/backups/etcd/pre-upgrade-*.db ~/
```

### 9.3 Back up /etc/kubernetes — [CONTROL PLANE]

```bash
sudo tar czf /var/backups/kubernetes-etc-$(date +%Y%m%d-%H%M%S).tar.gz -C / etc/kubernetes
```

**Why this is separate from the etcd snapshot:** the etcd snapshot holds
the cluster's *desired state* (Deployments, Services, etc.), but not the
PKI, static pod manifests, or kubeconfigs that live in `/etc/kubernetes`. If
that directory gets corrupted, the etcd snapshot alone can't recover you —
take both, every time.

**Verify:**
```bash
ls -lh /var/backups/kubernetes-etc-*.tar.gz
tar tzf /var/backups/kubernetes-etc-*.tar.gz | head -20
```
Expect roughly 30 KB–1 MB, listing `pki/`, `manifests/`, and the various
`.conf` files.

### OPTIONAL — back up /etc/kubernetes on each worker too

Workers have much less in there (mainly `kubelet.conf` and a couple of
certs), but it's cheap insurance.

```bash
sudo tar czf /var/backups/kubernetes-etc-$(date +%Y%m%d-%H%M%S).tar.gz -C / etc/kubernetes
```

### 9.4 Check capacity and what will move — [BASTION]

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=worker1
kubectl get pdb -A
```

**Why:** listing worker1's pods shows exactly what gets evicted when you
drain it — confirm they're Deployment replicas (which reschedule
elsewhere) rather than singleton or standalone pods (which don't).
`kubectl get pdb -A` should return nothing in this lab — that's fine as
long as there's enough spare capacity on the remaining workers.

### OPTIONAL — find standalone pods and emptyDir volumes

```bash
sudo apt-get install -y jq

# Pods with no owner (Deployment/ReplicaSet/etc.) — these are LOST on drain, not rescheduled
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.metadata.ownerReferences == null or (.metadata.ownerReferences | length == 0)) |
  .metadata.namespace + "/" + .metadata.name
'

# Pods using emptyDir volumes — the drain command needs --delete-emptydir-data for these
kubectl get pods -A -o json | jq -r '
  .items[] | select(.spec.volumes[]?.emptyDir != null) |
  .metadata.namespace + "/" + .metadata.name
'
```

### ✅ Phase 0 complete when:

- All health checks pass
- The etcd snapshot exists and its `snapshot status` shows real numbers
- The `/etc/kubernetes` tarball exists on the control plane
- You know exactly what will move during each worker's drain, and nothing
  critical is an unprotected singleton

---

## 10. If Something Goes Wrong — Rollback & Disaster Recovery

Read this once, before you start the actual upgrade, so you're not learning
it for the first time mid-incident. Keep it open in a second tab while you
work through Phases 1–5.

### First: figure out how bad it is

```bash
# From the bastion — is the API server reachable at all?
kubectl get nodes
kubectl get --raw='/readyz?verbose' | tail -10
```

| What you see | What it means | What to do |
|---|---|---|
| `kubectl get nodes` responds (even slowly), one node is `NotReady` or a pod is misbehaving | Likely a specific, fixable problem | Go to the matching entry in [Troubleshooting](#17-troubleshooting) — don't restore anything yet |
| `kubectl` can't reach the API server at all, control-plane pods are down and not recovering | Control plane is broken | Read **10.1** below |
| Only one worker is broken (won't rejoin, kubelet won't start, etc.) and the control plane is fine | Worker is broken, not the cluster | Read **10.2** below — usually far cheaper than restoring anything |

**General principle:** because this cluster has a single control plane, a
broken control plane is a full outage — there's no second CP to fail over
to. This is exactly why Phase 0's snapshot and tarball are non-negotiable,
and why a genuinely production cluster should have 3 control planes (see
the build doc's "Known Gaps" section).

### 10.1 Control plane is broken

**Step 1 — diagnose before restoring anything.** Restoring etcd is a last
resort, not a first response, because it discards any cluster changes made
since the snapshot.

```bash
# On the control plane
sudo crictl ps -a | grep -E 'etcd|apiserver|controller-manager|scheduler'
sudo crictl logs <container-id-of-the-broken-one>
sudo journalctl -u kubelet -n 100 --no-pager
```
Common, fixable causes at this stage: a bad image tag, a syntax error in a
static pod manifest under `/etc/kubernetes/manifests/`, a full disk, or a
certificate problem. Check those first — see
[17.8](#178-etcd-wont-start-after-upgrade) and
[17.9](#179-control-plane-pods-stuck-containercreating-after-upgrade).

**Step 2 — if etcd's data is actually corrupted and won't come back**, this
is the real recovery path:

```bash
# On the control plane
# 1. Stop kubelet so it doesn't keep restarting the broken etcd pod
sudo systemctl stop kubelet

# 2. Move the corrupted data directory aside — don't delete it yet
sudo mv /var/lib/etcd /var/lib/etcd.broken-$(date +%Y%m%d-%H%M%S)

# 3. Restore the most recent snapshot into a fresh data directory
sudo ETCDCTL_API=3 etcdctl snapshot restore /var/backups/etcd/<your-snapshot>.db \
  --name=controlplane \
  --initial-cluster=controlplane=https://<CP_PRIVATE_IP>:2380 \
  --initial-advertise-peer-urls=https://<CP_PRIVATE_IP>:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --data-dir=/var/lib/etcd

# 4. Restart kubelet — it will start etcd against the restored data
sudo systemctl start kubelet
```

**Verify:**
```bash
sudo crictl ps -a | grep etcd
# from the bastion
kubectl get nodes
kubectl get pods -A
```

**What you lose:** any cluster state changed between the snapshot and the
failure (new Deployments, scaled replicas, etc.) — another reason to take a
fresh snapshot immediately before starting Phase 1, not to rely on an old
one.

**Step 3 — if `/etc/kubernetes` itself is corrupted** (not just etcd —
e.g. the PKI or static manifests are damaged):

```bash
# On the control plane
sudo tar xzf /var/backups/kubernetes-etc-<timestamp>.tar.gz -C /
sudo systemctl restart kubelet
```

### 10.2 A worker is broken

Workers hold no unique cluster state — everything important lives in etcd
on the control plane. This means the cheapest fix for a broken worker is
almost always **reset and rejoin**, not repair:

```bash
# On the broken worker
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd
```
```bash
# On the control plane — get a fresh join command
sudo kubeadm token create --print-join-command
```
```bash
# On the worker — rejoin
sudo kubeadm join <CP_PRIVATE_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

**Verify (bastion):**
```bash
kubectl get nodes -o wide
```

### 10.3 Commands run on the wrong host (the most common real mistake)

If you accidentally ran node-upgrade commands on the **bastion** — e.g.
`kubelet`/`kubeadm` got installed there — see
[17.3](#173-wrong-host-mistake--commands-run-on-the-bastion) for the exact
cleanup, and consider applying the apt-pin lockdown in
[Phase 3](#13-phase-3--bastion-kubectl-upgrade--lockdown) to make this
mistake impossible going forward.

### 10.4 When to stop and not try to fix it yourself

If, after trying the above, the control plane still won't come back and
you don't have a recent snapshot: stop making changes, and rebuild the
cluster from the build doc rather than experimenting further on a broken
control plane — every additional change makes the eventual diagnosis
harder. This is a rare outcome if Phase 0 was actually completed.

---

## 11. Phase 1 — Control Plane Upgrade

Run all of this on the **control plane**, unless noted.

### 11.1 Point apt at the v1.33 repo

```bash
cat /etc/apt/sources.list.d/kubernetes.list
sudo sed -i 's|/v1.32/|/v1.33/|' /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
grep -r 'apt.kubernetes.io' /etc/apt/sources.list.d/ /etc/apt/sources.list 2>/dev/null
```

**Why:** Kubernetes repos are per-minor at `pkgs.k8s.io` — pointing apt at
`v1.33` is what makes 1.33 packages installable at all. This only touches
the control plane; workers stay on the v1.32 repo until their own turn in
Phase 2, which is what keeps version skew safe during the roll. The `grep`
just confirms there's no leftover reference to the old, deprecated
`apt.kubernetes.io` repo that could cause conflicting package sources.

**Verify:** the file's only line ends in `.../v1.33/deb/ /`.

### 11.2 Determine the exact target version

```bash
apt-cache madison kubeadm | head -10
```

**Why:** never hardcode a patch version from a doc — always check what's
actually available. Per the N-1 policy ([§5](#5-the-n-1-patch-policy)), pick
the *second* line, not the first, and note the full `<version>-<revision>`
string (e.g. `1.33.12-1.1`) — you'll reuse it in every install below.

### 11.3 Upgrade kubeadm on the control plane

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=<TARGET_VERSION>
sudo apt-mark hold kubeadm
kubeadm version
kubelet --version   # should still show the OLD version — expected
apt-mark showhold
```

**Why:** unhold → install the exact target → re-hold prevents `apt upgrade`
from drifting `kubeadm` again until you deliberately plan the next
upgrade. Upgrading `kubeadm` itself does not touch the running cluster —
it's just a CLI tool at this point.

### OPTIONAL — pre-pull images to reduce restart lag

```bash
sudo kubeadm config images pull --kubernetes-version v<TARGET_VERSION_NO_REVISION>
```
> ⚠️ Always pass `--kubernetes-version` explicitly here. Without it, kubeadm
> resolves the version from the remote "stable" channel and may pull the
> very latest patch instead of the N-1 target you chose — see
> [17.2](#172-kubeadm-config-images-pull-pulls-the-wrong-patch).

### 11.4 Plan, then apply the upgrade

```bash
sudo kubeadm upgrade plan v<TARGET_VERSION_NO_REVISION>
```
Review the output: target version matches what you chose, every component
lists a sensible current/target pair, and there are no preflight errors.

```bash
sudo kubeadm upgrade apply v<TARGET_VERSION_NO_REVISION>
```

**Why:** always pass the version to both `plan` and `apply` — without it,
kubeadm resolves the target from the remote channel and may suggest the
wrong patch (see [17.1](#171-kubeadm-upgrade-plan-resolves-the-wrong-version)).
`apply` rewrites the static pod manifests for `etcd`, `kube-apiserver`,
`kube-controller-manager`, and `kube-scheduler` with new image tags, renews
the certificates it manages by default, and rolls `CoreDNS` and
`kube-proxy`. Expect roughly 10–30 seconds of API server unavailability
during its own restart — workloads keep running throughout; only `kubectl`
and the controllers blip.

**Verify:** the output ends with
`SUCCESS! A control plane node of your cluster was upgraded to "v<TARGET>"`.

### 11.5 Upgrade kubelet on the control plane node

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=<TARGET_VERSION>
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo apt-mark hold kubelet
kubelet --version
```

**Why:** `kubeadm upgrade apply` only touches the static-pod control-plane
components — kubelet is a separate package, upgraded like on any node.
There's deliberately no `kubectl=...` in this install — this node never
gets `kubectl`.

**Verify (from the bastion):**
```bash
kubectl get nodes -o wide
kubectl version
```
The control plane should show the new version; workers still show the old
one — expected until Phase 2.

---

## 12. Phase 2 — Worker Upgrades (one at a time)

Repeat this entire phase once per worker. Never upgrade two workers
simultaneously.

### 12.1 Pre-checks

```bash
# On the bastion
kubectl get pods -A -o wide --field-selector spec.nodeName=worker1
kubectl get pdb -A
```
```bash
# On worker1 — always confirm you're on the right host first
hostname
cat /etc/apt/sources.list.d/kubernetes.list
kubeadm version
kubelet --version
apt-mark showhold
```

**Why the `hostname` check:** the single most common real mistake in this
kind of upgrade isn't a bad command — it's running the right command on
the wrong terminal tab. A quick `hostname` before any node-modifying block
catches it before it does damage.

### 12.2 Cordon and drain — [BASTION]

```bash
kubectl cordon worker1
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

**Why:** `cordon` stops new pods scheduling on the node while you inspect
what's there; `drain` then evicts existing pods through the Eviction API,
which respects any PDBs. DaemonSet pods are left alone
(`--ignore-daemonsets`) since they're meant to run on every node.

**Verify:** output ends with `node/worker1 drained`; the evicted pods
reappear `Running` on the remaining workers (`kubectl get pods -A -o wide`).
If the drain stalls with a PDB-violation error, see
[17.5](#175-drain-stalls-with-cannot-evict-pod-as-it-would-violate-the-pods-disruption-budget).

### 12.3 Upgrade kubeadm on the worker — [WORKER]

```bash
sudo sed -i 's|/v1.32/|/v1.33/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=<TARGET_VERSION>
sudo apt-mark hold kubeadm
kubeadm version
```

### 12.4 Run kubeadm upgrade node — [WORKER]

```bash
sudo kubeadm upgrade node
```

**Why:** this performs the node-local parts of the upgrade — kubelet
configuration and any local CNI config — bringing this worker in line with
the already-upgraded control plane. It doesn't touch the control plane
itself. Expect the output to end with something like *"The kubelet
configuration for this node was successfully upgraded!"*; a warning about
`KubeProxyConfiguration.bindAddress` is informational, not an error.

### 12.5 Upgrade kubelet on the worker — [WORKER]

```bash
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=<TARGET_VERSION>
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo apt-mark hold kubelet
kubelet --version
```

### 12.6 Uncordon and verify — [BASTION]

```bash
kubectl uncordon worker1
kubectl get nodes -o wide
```
**Verify:** the worker shows `Ready` and the new version.

Repeat 12.1 → 12.6 for worker2, then worker3.

### OPTIONAL — reboot each worker right after its own upgrade

It's simpler to do one clean reboot cycle at the very end (Phase 5). If
you'd rather reboot immediately after each worker instead, that's fine too
— just wait 2–3 minutes and re-check `kubectl get nodes` before starting
the next worker.

```bash
sudo reboot
```

---

## 13. Phase 3 — Bastion kubectl Upgrade + Lockdown

### 13.1 Upgrade kubectl — [BASTION]

```bash
sudo sed -i 's|/v1.32/|/v1.33/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-mark unhold kubectl
sudo apt-get update
sudo apt-get install -y kubectl=<TARGET_VERSION>
sudo apt-mark hold kubectl
kubectl version --client
kubectl version
```

**Why:** `kubectl` is technically allowed to be within ±1 minor of the
API server, so this could wait — but keeping it in lockstep avoids ever
having to think about client/server skew.

### OPTIONAL — permanently block kubelet/kubeadm from ever landing on the bastion

This directly prevents the wrong-host mistake described in
[17.3](#173-wrong-host-mistake--commands-run-on-the-bastion) — worth doing
once, on the bastion only.

```bash
sudo tee /etc/apt/preferences.d/kubernetes-bastion.pref <<EOF
Package: kubelet kubeadm kubernetes-cni
Pin: release *
Pin-Priority: -1
EOF
```
`Pin-Priority: -1` tells apt to refuse installing these packages here no
matter what — even an explicit `apt-get install kubelet` will fail. Confirm
this file exists **only** on the bastion, never on a cluster node — if
copied there by mistake it would block legitimate kubelet/kubeadm upgrades
on that node.

---

## 14. Phase 4 — Post-Upgrade Validation

Run from the **bastion**, unless noted.

### Cluster-wide health

```bash
kubectl get nodes -o wide
kubectl get ds -A
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
```
Expect all 4 nodes on the new version, full DaemonSet coverage
(`DESIRED = CURRENT = READY = AVAILABLE`), and nothing from the
field-selector query.

### Confirm component images actually moved

```bash
kubectl get ds -n kube-system kube-proxy -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get deploy -n kube-system coredns -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -n kube-system -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}' | sort
```

### Cross-node pod-to-pod networking (the real test)

```bash
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}'
```
```bash
kubectl run nettest --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# inside, curl each pod IP from the list above; expect 200 from all of them
```
This is the single most important verification — it confirms the data
plane, including cross-subnet VXLAN, survived the upgrade.

### DNS resolution

```bash
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
```

### NodePort from every worker

```bash
NP=$(kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}')
for ip in <worker1-ip> <worker2-ip> <worker3-ip>; do
  echo -n "$ip -> "; curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 -I http://$ip:$NP
done
```

### etcd and API server health

```bash
kubectl get --raw='/readyz?verbose' | tail -5
kubectl -n kube-system exec etcd-controlplane -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint health
```

### ✅ Phase 4 complete when:

All nodes report the new version, all pods `Running`, cross-node traffic
and DNS both work, NodePort responds from every worker, and both `/readyz`
and `etcd endpoint health` report healthy.

---

## 15. Phase 5 — Reboot Cycle

Recommended after every Kubernetes upgrade — it forces clean restarts of
kubelet, containerd, and networking, and surfaces any gaps immediately
rather than at some unrelated moment later.

**Control plane first, then each worker, one at a time:**
```bash
sudo reboot
```
Wait 5–10 minutes, then from the bastion:
```bash
kubectl get nodes
kubectl get pods -A
```
Confirm the rebooted node is `Ready` and everything is `Running` before
moving to the next node.

**What to expect:** every pod shows `RESTARTS 1` at the reboot's timestamp
— that's correct, not a crash loop (see
[17.6](#176-pods-show-restarts-1-after-the-reboot-cycle)).

---

## 16. Phase 6 — Cleanup

```bash
# On each node
apt-mark showhold
```
```bash
# On the control plane — decide, per your own retention policy, whether to
# keep or remove the pre-upgrade backups
ls -lh /var/backups/etcd/ /var/backups/kubernetes-etc-*.tar.gz
```
Update this document (or your own notes) with the version you actually
landed on and the date, so the next upgrade has a clear starting point —
see the [Closing Notes](#closing-notes) table below for the format.

---

## 17. Troubleshooting

### 17.1 `kubeadm upgrade plan` resolves the wrong version

**Symptom:** the plan targets a newer patch than the one you chose (e.g.
`1.33.13` when you wanted `1.33.12`).

**Cause:** `kubeadm upgrade plan` with no version argument fetches "latest
stable in the current minor" from the remote version API — that's the
absolute latest patch, not your N-1 target.

**Fix:** always pass the version explicitly:
```bash
sudo kubeadm upgrade plan v1.33.12
sudo kubeadm upgrade apply v1.33.12
```

### 17.2 `kubeadm config images pull` pulls the wrong patch

**Symptom:** logs show it pulled a newer patch than intended.

**Cause:** same root cause as 17.1 — without `--kubernetes-version` it
resolves from the remote channel.

**Fix:**
```bash
sudo kubeadm config images pull --kubernetes-version v1.33.12
```

### 17.3 Wrong-host mistake — commands run on the bastion

**Symptom:** a worker-upgrade block was accidentally run on the bastion;
`kubelet`/`kubeadm`/`kubernetes-cni` got installed there.

**Fix:**
```bash
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get remove -y --purge kubelet kubeadm kubernetes-cni
sudo apt-get autoremove -y
dpkg -l | grep -E 'kubelet|kubeadm|kubectl|kubernetes-cni'   # should show only kubectl
sudo apt-mark hold kubectl
```
**Prevention:** run `hostname` before every node-modifying block, and apply
the apt-pin lockdown from [Phase 3](#13-phase-3--bastion-kubectl-upgrade--lockdown).

### 17.4 `kubeadm upgrade node` fails with "no kubelet.conf"

**Symptom:**
```
couldn't create a Kubernetes client from file "/etc/kubernetes/kubelet.conf":
open /etc/kubernetes/kubelet.conf: no such file or directory
```
**Cause:** run on a host that isn't an actual cluster node (e.g. the
bastion). **Fix:** run it on the real node — this error is a safety net
that prevented it from doing anything on the wrong host.

### 17.5 Drain stalls with "Cannot evict pod as it would violate the pod's disruption budget"

**Cause:** a PDB is blocking eviction because it would drop below its
required minimum.

**Fix, in order of preference:**
1. Free up capacity so a replacement pod can schedule and become `Ready`
   elsewhere.
2. Temporarily scale up (`kubectl scale deploy/web --replicas=8`), drain,
   then scale back down afterward.
3. `kubectl describe pod <replacement>` — maybe it's stuck `Pending` or
   failing readiness, so it never becomes `Ready` and never unblocks the
   next eviction.
4. Relax the PDB — last resort, and only temporarily.

### 17.6 Pods show `RESTARTS 1` after the reboot cycle

**Not a crash loop.** Rebooting a node restarts kubelet, which restarts
every pod on it exactly once. A genuine crash loop looks like a high,
still-incrementing restart count with a recent timestamp — a single
restart matching the reboot time is healthy.

### 17.7 Pods `Pending` after the upgrade

```bash
kubectl describe pod <name> | tail -20
```
The Events section names the exact reason (insufficient capacity, node
selector mismatch, a taint, etc.).

### 17.8 etcd won't start after upgrade

```bash
sudo crictl ps -a | grep etcd
sudo crictl logs <etcd-container-id>
sudo journalctl -u kubelet -n 100 --no-pager
```
Common causes: certificate mismatch, a full disk, or a corrupted data
directory. If it's genuinely corrupted, see
[10.1](#101-control-plane-is-broken) for the full snapshot-restore
procedure.

### 17.9 Control-plane pods stuck `ContainerCreating` after upgrade

```bash
sudo crictl ps -a | grep -E 'kube-apiserver|controller-manager|scheduler'
sudo crictl logs <container-id>
sudo journalctl -u kubelet -n 100 --no-pager
```
Common causes: wrong image tag, an image pull failure, a certificate issue,
or a syntax error in the static pod manifest.

### 17.10 `calico-node` readiness drops after the upgrade

Same root causes as the initial build — BGP left enabled, or Typha (TCP
5473) unreachable:
```bash
kubectl get installation.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'
```
If not `Disabled`:
```bash
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node
```

### 17.11 "remote version is much newer: v1.37.0" during a kubeadm command

**Informational only** — kubeadm is telling you the latest Kubernetes
release in the wild while correctly limiting itself to the minor you
targeted. No action needed.

### 17.12 "Pending kernel upgrade" warning

**Cause:** a kernel package updated but the host hasn't rebooted yet.
**Fix:** ignore it during the upgrade itself; the Phase 5 reboot cycle
picks it up. Don't reboot mid-upgrade, only at the planned point.

### 17.13 `apt-get autoremove` wants to remove `cri-tools`

**Cause:** apt thinks nothing depends on it, but `cri-tools` provides
`crictl`, which you use constantly for debugging.

**Fix:**
```bash
sudo apt-mark manual cri-tools
```
Do this before ever running `apt-get autoremove` on a node.

### 17.14 The bastion's apt pin blocks a legitimate install — on a node

**Cause:** the `Pin-Priority: -1` file from Phase 3 was copied onto a
cluster node by mistake, where it now blocks real kubelet/kubeadm updates.

**Fix:** confirm the pin file only exists on the bastion:
```bash
ls /etc/apt/preferences.d/kubernetes-bastion.pref
```
Remove it from any node it shouldn't be on.

---

## 18. Lessons Learned From This Upgrade

- **`etcdctl` isn't preinstalled** — budget for `apt-get install -y
  etcd-client` in Phase 0, every time.
- **`kubeadm upgrade plan`, `apply`, and `config images pull` all silently
  resolve to the latest remote patch unless you pass the version
  explicitly.** Always type the version out.
- **The most common real mistake is running the right command on the wrong
  host**, not a bad command — `hostname` before every node-modifying block,
  and the bastion apt-pin lockdown, both exist specifically to catch this.
- **Confirm Phase 0's outputs before moving on**, not just "looks good" —
  skipping straight past the snapshot/backup verification is the exact gap
  that leaves you with no rollback if the control-plane upgrade goes wrong.
- **`apt-cache madison kubeadm` is the source of truth for the target
  version** — don't hardcode a patch from a doc; check what's actually
  available, then apply the N-1 policy on top of that.
- **The whole cluster stays healthy through the roll** if you go one node
  at a time and have spare replica capacity — CoreDNS and workload pods
  simply reschedule onto whichever workers are still up.

---

## 19. Appendix: Ports, Paths, Reference Commands

### Ports and protocols

| Component | Port | Protocol | Purpose |
|---|---|---|---|
| kube-apiserver | 6443 | TCP | API server |
| etcd client | 2379 | TCP | etcd API |
| etcd peer | 2380 | TCP | etcd replication |
| kubelet | 10250 | TCP | kubelet API |
| kube-scheduler | 10259 | TCP | metrics/serving |
| kube-controller-manager | 10257 | TCP | metrics/serving |
| NodePort services | 30000–32767 | TCP | default NodePort range |
| Calico Typha | 5473 | TCP | Calico datastore proxy |
| **Calico VXLAN** | **4789** | **UDP** | **VXLAN encapsulation** |
| Calico BGP | 179 | TCP | BGP (disabled in this setup) |
| CoreDNS | 53 | UDP + TCP | cluster DNS |

**Golden rule:** always specify both protocol and port. `4789` alone is
meaningless; `UDP 4789` is what VXLAN actually uses.

### Key paths

| Path | Content |
|---|---|
| `/etc/kubernetes/` | all cluster config + PKI |
| `/etc/kubernetes/manifests/` | static pod manifests (etcd, apiserver, scheduler, controller-manager) |
| `/etc/kubernetes/pki/` | all certificates |
| `/etc/kubernetes/admin.conf` | cluster-admin kubeconfig |
| `/etc/kubernetes/kubelet.conf` | kubelet's kubeconfig |
| `/var/lib/kubelet/` | kubelet state |
| `/var/lib/etcd/` | etcd data directory |
| `/var/backups/etcd/` | this cluster's etcd snapshots |
| `/var/backups/kubernetes-etc-*.tar.gz` | this cluster's `/etc/kubernetes` backups |
| `/etc/apt/sources.list.d/kubernetes.list` | apt repo pointer for the Kubernetes minor |
| `/etc/apt/preferences.d/kubernetes-bastion.pref` | bastion lockdown — should exist only on the bastion |

### Quick reference commands

```bash
# Current versions, everywhere it matters
kubectl version
kubeadm version
kubelet --version
containerd --version
etcdctl version

# Node / component state
kubectl get nodes -o wide
kubectl describe node <name>
kubectl get pods -n kube-system -o wide
kubectl get ds -A

# What's pinned right now
apt-mark showhold

# Container runtime debugging (on any node)
sudo crictl ps -a
sudo crictl images
sudo crictl logs <container-id>

# etcd health (on the control plane)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# etcd snapshot (on the control plane)
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd/$(date +%Y%m%d-%H%M%S).db
```

---

## 20. References

- Kubernetes — Version Skew Policy: https://kubernetes.io/releases/version-skew-policy/
- Kubernetes — Releases and Support Policy: https://kubernetes.io/releases/
- Kubernetes — Upgrading kubeadm Clusters: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
- Kubernetes — Change Package Repository: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/
- Kubernetes — Disruptions (PodDisruptionBudget): https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
- Tigera Calico — Kubernetes Compatibility Matrix: https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
- containerd — Releases: https://containerd.io/releases/

---

## Closing Notes

### Where this leaves the cluster

| Component | Version |
|---|---|
| Kubernetes (all nodes) | v1.33.12 |
| kubeadm (all nodes) | v1.33.12 |
| kubelet (all nodes) | v1.33.12 |
| kubectl (bastion) | v1.33.12 |
| etcd | 3.5.24-0 |
| CoreDNS | v1.12.0 |
| kube-proxy | v1.33.12 |
| Calico | v3.30.2 |
| containerd | 2.2.1 |
| OS | Ubuntu 24.04.4 LTS |

**Next upgrade target:** 1.34.x, once it reaches N-1 status (roughly 4
months after 1.34 becomes current).

### What I'd do differently next time

- Add PDBs and readiness probes to real workloads before ever draining a
  node running them — this lab got away without them due to spare
  capacity, production shouldn't rely on that.
- Have a written maintenance window with an announced start/end time and a
  rollback decision already made, rather than deciding reactively.
- Automate Phase 0 as a single script that prints a clear green/red per
  check, so pre-flight can't be skipped by accident.
- Take the etcd snapshot off-box (S3 or another host) as standard practice,
  not an optional step, given this cluster has only one control plane.

---
