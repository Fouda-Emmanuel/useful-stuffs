# Kubernetes etcd Restore from Snapshot: Complete Runbook

A hands-on, step-by-step guide to restoring a **3-member stacked etcd cluster** on a kubeadm-based HA Kubernetes cluster from a snapshot (sourced from S3), including point-in-time proof, post-restore worker troubleshooting, and production guidance.

Written from a real drill on the `faekcorp` lab. Every command below was run in that lab unless marked *(not run)*.

| Item | Value |
|---|---|
| Kubernetes | v1.35.x (kubeadm), containerd, Cilium CNI |
| etcd | 3.6.6 (`registry.k8s.io/etcd:3.6.6-0`) |
| Cluster name / domain | `faekcorp-lab` / `faekcorp.lab` |
| Nodes | `cp01`, `cp02`, `cp03` (stacked etcd) + `worker1`, `worker2` |
| Load balancer | HAProxy at `haproxy.faekcorp.lab:6443` |
| Admin host | Bastion (kubectl-only); control planes reachable only through it |
| Snapshot bucket | `s3://ha-cluster-s3-etcd-lab/etcd-snapshots/` (us-east-1) |
| Companion doc | *Kubernetes etcd Backup & S3 Upload: Complete Runbook* |

> **Adapt before use.** Hostnames, IPs, bucket, snapshot filename, and the bastion address are lab values. Replace them with yours. Never publish real public IPs or key names in a public repo (the bastion IP is shown as `<BASTION_PUBLIC_IP>`).

---

## Table of Contents

**Part 1: The Mental Model**
1. [Restore is a build operation, not a file copy](#1-restore-is-a-build-operation-not-a-file-copy)
2. [Do all three members hold the same data?](#2-do-all-three-members-hold-the-same-data)
3. [The three different "snapshots"](#3-the-three-different-snapshots)
4. [Why restore creates a new cluster identity](#4-why-restore-creates-a-new-cluster-identity)
5. [Why all members must be restored from the same snapshot](#5-why-all-members-must-be-restored-from-the-same-snapshot)
6. [The restore flags: shared vs per-node](#6-the-restore-flags-shared-vs-per-node)
7. [Why the manifests show different `--initial-cluster` values](#7-why-the-manifests-show-different---initial-cluster-values)
8. [What an etcd restore does and does not cover](#8-what-an-etcd-restore-does-and-does-not-cover)

**Part 2: Before You Touch Anything**

9.  [Pre-flight checklist](#9-pre-flight-checklist)
10. [Point-in-time behavior: what survives, what disappears](#10-point-in-time-behavior-what-survives-what-disappears)
11. [Create canary resources](#11-create-canary-resources)
12. [etcd 3.6 version notes](#12-etcd-36-version-notes)


**Part 3: Sourcing the Snapshot**

13. [Local file vs S3 vs laptop hop](#13-local-file-vs-s3-vs-laptop-hop)
14. [Path A: Download from S3](#14-path-a-download-from-s3)
15. [Path B: Copy via admin laptop through the bastion](#15-path-b-copy-via-admin-laptop-through-the-bastion)
16. [The `~/s3-etcd-backups/` convention](#16-the-s3-etcd-backups-convention)

**Part 4: The Restore Procedure**

17. [Stage A: Save the current manifest state](#17-stage-a-save-the-current-manifest-state)
18. [Stage B: Stop kubelet and the control-plane containers](#18-stage-b-stop-kubelet-and-the-control-plane-containers)
19. [Stage C: Preserve the current data directories](#19-stage-c-preserve-the-current-data-directories)
20. [Stage D: Restore on each node](#20-stage-d-restore-on-each-node)
21. [Stage E: Fix permissions, start kubelet](#21-stage-e-fix-permissions-start-kubelet)
22. [Stage F: Watch quorum reform](#22-stage-f-watch-quorum-reform)

**Part 5: Validation**

23. [etcd cluster health](#23-etcd-cluster-health)
24. [Proving point-in-time correctness](#24-proving-point-in-time-correctness)
25. [Workload and scheduling tests](#25-workload-and-scheduling-tests)

**Part 6: After the Restore: Things That Go Wrong**

26. [`cilium-operator` CrashLoopBackOff (Lease-based controllers)](#26-cilium-operator-crashloopbackoff-lease-based-controllers)
27. [Troubleshooting case: stale containers and Pending Pods on workers](#27-troubleshooting-case-stale-containers-and-pending-pods-on-workers)

**Part 7: Cleanup and Production**

28. [Cleanup: what to keep and when to delete](#28-cleanup-what-to-keep-and-when-to-delete)
29. [Production considerations](#29-production-considerations)

**Part 8: Reference**

30. [Complete command reference](#30-complete-command-reference)
31. [Key paths](#31-key-paths)
32. [Troubleshooting table](#32-troubleshooting-table)
33. [One-page flow and quick sequence](#33-one-page-flow-and-quick-sequence)
34. [Glossary](#34-glossary)

---

# Part 1: The Mental Model

## 1. Restore is a build operation, not a file copy

Backup is a **read** operation. Restore is a **build** operation.

- `etcdctl snapshot save` produces a **portable logical dump**: every key, every value, revision numbers, membership, and the cluster ID at snapshot time. It is **not** a copy of `/var/lib/etcd/`.
- `etcdutl snapshot restore` reads that dump and **builds a fresh etcd data directory** that looks like a just-bootstrapped member, pre-loaded with the snapshot's data. It also assigns a **new cluster ID**, **new member IDs**, and writes a **new membership list** (from the `--initial-cluster` flag you supply).

```
        BACKUP                                    RESTORE
        ──────                                    ───────
  live cluster                           portable snapshot file
       │                                          │
       ▼                                          ▼
 etcdctl snapshot save                   etcdutl snapshot restore
       │                                          │
       ▼                                          ▼
 one portable .db file                   fresh etcd data dir
 (dated, uploaded to S3)                 (member/snap/db + wal + NEW cluster-id)
```

> The snapshot is the **brain**. `--data-dir` is where the new **body** is built. `--name` and `--initial-advertise-peer-urls` tell the body **who it is**.

Why `--data-dir=/var/lib/etcd` and why the old directory must be moved first: restore is a directory builder. It will not write into an already-populated directory, so the old one must be moved aside (Stage C).

## 2. Do all three members hold the same data?

**Logically yes, physically no.** Every write goes through Raft: the leader appends, replicates, and an entry is committed once a majority (2 of 3) acknowledge. All members end up with identical keys, values and revisions. But each member has its **own independent copy** on its own disk (`/var/lib/etcd/member/`), with its own WAL and db file. The bytes differ; the logical state is the same.

```
         Logical state (same on all 3)
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
    cp01          cp02          cp03
 /var/lib/etcd /var/lib/etcd /var/lib/etcd   (3 physical copies)
```

**Can the snapshot be taken from any member?** Yes. Any healthy member can serve it; the content reflects the cluster's committed state, not which member you asked. We used cp01 only for convenience (certs and tools were there).

## 3. The three different "snapshots"

| Name | Path | What it is | In our backup file? |
|---|---|---|---|
| Internal checkpoint | `/var/lib/etcd/member/snap/*.snap` | etcd's own Raft metadata checkpoints (~9 KB each) | No |
| Live database | `/var/lib/etcd/member/snap/db` | The live bbolt DB file | Contents yes, the file no |
| **Backup snapshot** | file produced by `snapshot save` | Portable logical dump | **Yes, this is our file** |

Only the third is used for disaster recovery. Why not just copy `member/snap/db`? It is written live (risk of a torn copy), it misses un-checkpointed WAL entries, its format is internal, and there is no supported restore path from it.

The snapshot does **not** contain WAL files, internal checkpoints, file permissions, disk paths, or per-node identity (which node you are restoring onto).

## 4. Why restore creates a new cluster identity

A restored data directory has no memory of the old cluster ID. `etcdutl` writes a **new** cluster ID into the fresh directory. A member with a new cluster ID refuses to talk to members with the old one. This is by design: it prevents a restored member from accidentally merging into a partially-live cluster.

## 5. Why all members must be restored from the same snapshot

If you restore only cp01:

```
cp01 (restored):   "I'm in cluster XYZ, peers are A/B/C"
cp02 (untouched):  "I'm in cluster ABC, peers are A/B/C"
cp03 (untouched):  "I'm in cluster ABC, peers are A/B/C"
```

cp01 and cp02/cp03 refuse to talk (different cluster IDs). You get **split brain**: a lone restored member, plus two members that still hold the bad data.

Raft replicates log entries *between members of the same cluster*. It cannot change a member's cluster ID, merge two clusters, or accept a member with a different cluster ID. The only way to get all three into one new cluster is to restore **all three** from the **same snapshot** with an **identical `--initial-cluster`**.

**Analogy:** three friends share a notebook for "Club ABC". One page is corrupted and you want to roll back to page 500. You can't give one friend a fresh notebook called "Club XYZ" and expect the others to copy it, because they only recognize Club ABC. You give **all three** the same new notebook, and tell each who they are and who the other members are.

## 6. The restore flags: shared vs per-node

| Flag | Same on all 3? | Why |
|---|---|---|
| `--name` | No (per node) | Each member has its own name |
| `--initial-cluster` | **Yes, byte-identical** | The membership contract; every node must agree on who is in the cluster |
| `--initial-advertise-peer-urls` | No (per node) | Each member advertises its own peer URL |
| `--data-dir` | Yes | `/var/lib/etcd` on every node |

The shared `--initial-cluster` string for this lab:

```text
cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380
```

## 7. Why the manifests show different `--initial-cluster` values

On a kubeadm cluster, `/etc/kubernetes/manifests/etcd.yaml` may show cp01 with only cp01, cp02 with cp01+cp02, and cp03 with all three. That is a leftover from how each node *joined* and is **not** a problem.

`--initial-cluster` is only used the **first time** a node bootstraps an **empty** data directory. On normal restarts etcd reads membership from its populated data dir and ignores the flag.

During a restore the data dir **is empty**, so the value you pass to `etcdutl snapshot restore` is what gets written. It must be the **full, current, identical** member list on every node, not the stale per-node value from each manifest.

| Context | `--initial-cluster` value |
|---|---|
| Fresh `kubeadm init` (first node) | just that node |
| Fresh `kubeadm join` (new node) | whatever kubeadm passes (often partial) |
| **Restore** | **the full list, identical on all nodes** |

## 8. What an etcd restore does and does not cover

An etcd restore recovers **Kubernetes API state** (control-plane state). It is **not** a complete snapshot of every worker's container runtime, PersistentVolume data, container images, or cloud resources.

| Covered | Not covered |
|---|---|
| All Kubernetes objects (Deployments, Secrets, ConfigMaps, ...) | Containers already running on worker nodes (see [section 27](#27-troubleshooting-case-stale-containers-and-pending-pods-on-workers)) |
| Cluster membership of etcd | PV data, external databases, cloud load balancers, DNS |
| | Anything created outside Kubernetes after the snapshot (may become orphaned) |

---

# Part 2: Before You Touch Anything

## 9. Pre-flight checklist

| # | Check | Command | Expected |
|---|---|---|---|
| 1 | Snapshot present on all 3 CPs | `ls -lh ~/s3-etcd-backups/` | Dated file, ~12 MB |
| 2 | Snapshot integrity identical | `etcdutl snapshot status <path> -w table` | Same HASH on all 3 |
| 3 | etcd healthy | `etcdctl member list`, `endpoint health` | 3 started, all true |
| 4 | Topology/peer URLs confirmed | `sudo grep -E -- '--name=\|--initial-advertise-peer-urls=' /etc/kubernetes/manifests/etcd.yaml` | Matches known peer URLs |
| 5 | Current state saved | `kubectl get all -A -o wide > ~/pre-restore-cluster-state.txt` | File exists |
| 6 | Disk space | `df -h /var/lib/etcd/` | Snapshot size + headroom |
| 7 | SSH to all 3 CPs | `ssh ubuntu@cp02.faekcorp.lab` | Works (or use bastion/laptop hop) |
| 8 | Nothing already flapping | `kubectl get pods -n kube-system` | No unexplained restarts |

Details for the key checks:

**Record the "originals" (resources that must survive):**

```bash
kubectl get nodes
kubectl get namespaces
kubectl get deployments -A
kubectl get pods -A
kubectl get all -A -o wide > ~/pre-restore-cluster-state.txt
kubectl get ns            > ~/pre-restore-namespaces.txt
```

In our lab the originals were: namespaces `cilium-secrets`, `default`, `kube-node-lease`, `kube-public`, `kube-system`; deployments `nginx-deploy` (default), `cilium-operator` and `coredns` (kube-system); services `kubernetes`, `nginx-svc`, `cilium-envoy`, `hubble-peer`, `kube-dns`; DaemonSets `cilium`, `cilium-envoy`.

**Check the clock vs the snapshot time.** Our snapshot was taken `2026-10-06 22:33:31`. Anything older survives; anything newer vanishes.

```bash
date
```

**Verify snapshot integrity (offline, safe):**

```bash
etcdutl snapshot status ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db -w table
```

```text
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| fc420e4c |   214775 |        387 |      12 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

Write down **HASH** and **REVISION**. The HASH **must be identical on all three nodes**; that is your proof they restore from the same snapshot.

**Verify etcd is healthy before you break it:**

```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health -w table
```

Sample member list before restore (note the member IDs; they will change):

```text
| 574699e63127f8c0 | started | cp03.faekcorp.lab | https://10.70.41.250:2380 | https://10.70.41.250:2379 | false |
| 6108a621ed6af19b | started | cp01.faekcorp.lab |   https://10.70.21.6:2380 |   https://10.70.21.6:2379 | false |
| ee6b7ea6225a4e3e | started | cp02.faekcorp.lab | https://10.70.31.209:2380 | https://10.70.31.209:2379 | false |
```

Either `server.crt/server.key` or `healthcheck-client.crt/key` works for read-only checks. Be consistent.

## 10. Point-in-time behavior: what survives, what disappears

| Category | After restore |
|---|---|
| Resources created **before** the snapshot | Survive |
| Resources created **after** the snapshot | **Vanish** |
| Modifications made after the snapshot | **Reverted** |
| Deletions made after the snapshot | **Restored** |

Restore is an atomic rollback to the snapshot's exact state. It is not a merge and not a partial replay.

## 11. Create canary resources

Create resources **after** the snapshot moment. They exist in live etcd but not in the snapshot, so they must vanish after a correct restore. That is the proof.

```bash
kubectl create namespace sim-backup
kubectl create namespace after-snapshot

kubectl create deploy sim-deploy            --image=nginx --replicas 2 -n sim-backup
kubectl create deploy after-snapshot-deploy --image=nginx --replicas 2 -n after-snapshot

kubectl get deploy -n sim-backup
kubectl get deploy -n after-snapshot
```

Wait for `2/2` ready in both.

## 12. etcd 3.6 version notes

- `etcdctl snapshot restore` was **removed** in 3.6. Use **`etcdutl snapshot restore`** (offline, needs no certs).
- `ETCDCTL_API=3` is no longer needed. Setting it prints a harmless `unrecognized environment variable` warning.
- The snapshot `VERSION` column (e.g. `3.6.0`) is the snapshot format version and may differ from the binary version (e.g. `3.6.6`). Normal.

---

# Part 3: Sourcing the Snapshot

Every member needs the **same** snapshot file, staged and hash-verified **before** you stop anything. Never depend on the network while etcd is down.

## 13. Local file vs S3 vs laptop hop

| Source | Use when | Pros | Cons |
|---|---|---|---|
| Local file on cp01 | Snapshot already on the box; quick drill | No network dependency | If cp01's disk is lost, the copy is gone |
| **S3** | Real disaster; cp01 rebuilt; prove the full loop | Durable, off-site, the production path | Needs IAM role + AWS CLI + network |
| Laptop hop via bastion | CPs can't SSH each other and/or can't reach S3 | Works in hardened topologies | Needs laptop + bastion + keys |

| Aspect | Path A (S3) | Path B (laptop hop) |
|---|---|---|
| Works when CPs can't SSH each other | Yes | Yes |
| Works when CPs can't reach the internet | No | Yes |
| Requires IAM role on each CP | Yes | No |
| Requires bastion + laptop SSH keys | No | Yes |
| Preferred in a real disaster | **Yes** | Fallback |

In a real disaster you will almost always pull from S3, because the local copy would have died with the node. Practice that path. In our lab we documented both; Path B was the working path for cp02/cp03 during the drill.

## 14. Path A: Download from S3

On **each** CP:

```bash
# 1. Confirm credentials come from an IAM role (assumed-role ARN, NOT a static user)
aws sts get-caller-identity

# 2. Confirm the bucket/prefix is reachable
aws s3 ls s3://ha-cluster-s3-etcd-lab/etcd-snapshots/

# 3. Create the staging directory
mkdir -p ~/s3-etcd-backups

# 4. Download, keeping the original dated filename (give the directory, not a filename)
aws s3 cp s3://ha-cluster-s3-etcd-lab/etcd-snapshots/etcd-snapshot-cp01-20261006-223331.db \
  ~/s3-etcd-backups/

# 5. Verify
ls -lh ~/s3-etcd-backups/
etcdutl snapshot status ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db -w table
```

Expected on all three nodes: HASH `fc420e4c`, REVISION `214775`, TOTAL KEYS `387`. **If any node differs, stop and re-download.**

Optional cross-check (catches transfer corruption):

```bash
sha256sum ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db   # identical on all 3
```

If `aws sts get-caller-identity` fails with "Unable to locate credentials" on cp02/cp03, either attach the same IAM role to those instances or use Path B. Static keys in `~/.aws/credentials` should **not** exist on control planes; use an instance role.

## 15. Path B: Copy via admin laptop through the bastion

In this lab the CPs are on different segments and cannot SSH to each other. A direct copy fails:

```text
ubuntu@cp01:~$ scp etcd-snapshot-cp01-20261006-223331.db ubuntu@10.70.1.6:~
ubuntu@10.70.1.6: Permission denied (publickey).
scp: Connection closed
```

Hop through the admin laptop, tunnelling through the bastion with `ProxyCommand`. Two keys are involved: the **control-plane key** (destination auth) and the **bastion key** (jump host auth).

**Step 1: Pull the snapshot from cp01 to the laptop:**

```bash
scp -o IdentitiesOnly=yes \
    -i ~/.ssh/lab-controlplane-keypair.pem \
    -o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem -o IdentitiesOnly=yes -W %h:%p ubuntu@<BASTION_PUBLIC_IP>" \
    ubuntu@10.70.21.6:~/etcd-snapshot-cp01-20261006-223331.db \
    ~/
```

**Step 2: Push to cp02:**

```bash
scp -o IdentitiesOnly=yes \
    -i ~/.ssh/lab-controlplane-keypair.pem \
    -o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem -o IdentitiesOnly=yes -W %h:%p ubuntu@<BASTION_PUBLIC_IP>" \
    ~/etcd-snapshot-cp01-20261006-223331.db \
    ubuntu@10.70.31.209:~/
```

**Step 3: Push to cp03:** same command with `ubuntu@10.70.41.250:~/`.

Sample output (first connection asks to trust the host key; answer `yes`):

```text
etcd-snapshot-cp01-20261006-223331.db    100%   12MB 448.3KB/s   00:26
```

| Flag | Meaning |
|---|---|
| `-i ~/.ssh/lab-controlplane-keypair.pem` | Key that authenticates to the destination CP |
| `-o IdentitiesOnly=yes` | Use only the key given with `-i`, ignore ssh-agent (avoids "too many authentication failures") |
| `-o ProxyCommand="ssh -i <bastion-key> ... -W %h:%p ubuntu@<bastion>"` | Connect to the bastion and forward raw TCP to the destination host:port |

**Step 4: Verify on each CP** (file lands in the home dir):

```bash
ls -lh ~/etcd-snapshot-cp01-20261006-223331.db
etcdutl snapshot status ~/etcd-snapshot-cp01-20261006-223331.db -w table
```

HASH must be `fc420e4c` on all three. If Path B was used, the restore commands in Stage D use `~/etcd-snapshot-cp01-20261006-223331.db` as the source instead of the S3 folder path.

## 16. The `~/s3-etcd-backups/` convention

Two folders, two roles:

```text
~/etcd-snapshot-*.db        ← created during BACKUP, then uploaded to S3
~/s3-etcd-backups/*.db      ← downloaded from S3, ready for RESTORE
```

As snapshots accumulate, the S3 folder holds dated files and you choose one:

```bash
ls -lh ~/s3-etcd-backups/
# etcd-snapshot-cp01-20261006-223331.db
# etcd-snapshot-cp01-20261009-120000.db   <- newer
```

Rules: **keep the original dated filename** (never rename to `snapshot.db`, which hides the date and risks overwriting), never overwrite a snapshot, and restore using the full dated path.

---

# Part 4: The Restore Procedure

> **Window:** the API is **down** from Stage B until quorum reforms in Stage F (expect roughly 5 to 10 minutes; see [RTO](#29-production-considerations)). Do not start Stage B until: the snapshot is staged and hash-verified on all three CPs, you have **three SSH sessions open** (one per CP), and you are ready to run B to E in one sitting.
>
> **Order:** do each stage on **all three** nodes before moving to the next (all-C, then all-D, then all-E). Do not start kubelet on one node while the others are mid-restore.

## 17. Stage A: Save the current manifest state

Read-only safety net. On **each** CP:

```bash
sudo cp /etc/kubernetes/manifests/etcd.yaml ~/etcd.yaml.before-restore
sudo ls -l /etc/kubernetes/pki/etcd/ > ~/etcd-certs.before-restore.txt
```

If anything looks different after the restore, you can compare.

## 18. Stage B: Stop kubelet and the control-plane containers

On **each** CP (three terminals):

```bash
sudo systemctl stop kubelet
sudo systemctl is-active kubelet        # expect: inactive
```

**Why:** kubelet runs etcd as a *static pod*. If you modify `/var/lib/etcd` while kubelet is alive, it immediately restarts etcd and rewrites the directory. Stop kubelet on **all three**; if you stop only one, the other two keep quorum and the one you are working on tries to rejoin, causing confusion.

Wait ~30 seconds, then verify:

```bash
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler' || echo "all gone"
```

### Gotcha: static pod containers often do NOT stop

In our lab, 4+ minutes after `systemctl stop kubelet` the etcd and apiserver containers were still `Running`. Kubelet's static pod shutdown grace period can be long, and in some kubeadm setups the containers don't get a clean stop signal. Force-stop them **by container ID**.

> `crictl stop <pod-name>` (e.g. `etcd-cp01.faekcorp.lab`) is unreliable: it returned an ID but did not stop the container. Use the **container ID**.

```bash
# 1. List (leftmost column is the container ID)
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler'
```

```text
00ef7bbb9b6ca  ...  Running  kube-apiserver           7  ...  kube-apiserver-cp01.faekcorp.lab
85dc83466918a  ...  Running  kube-scheduler           9  ...  kube-scheduler-cp01.faekcorp.lab
441fce6d4945b  ...  Running  kube-controller-manager  8  ...  kube-controller-manager-cp01.faekcorp.lab
71f1a8c495cf5  ...  Running  etcd                     7  ...  etcd-cp01.faekcorp.lab
```

```bash
# 2. Stop by ID (one shot is fine)
sudo crictl stop <id-apiserver> <id-etcd> <id-controller-manager> <id-scheduler>

# 3. Verify; must print "all gone"
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler' || echo "all gone"

# 4. Any still running after ~10s? Force remove
sudo crictl rm -f <container-id>

# 5. Confirm nothing is bound to the etcd ports and no etcd process exists
sudo ss -tlnp | grep -E '2379|2380' || echo "ports 2379/2380 free"
sudo ps aux | grep -E '[e]tcd' || echo "no etcd process"
```

Both must be clean on every CP. A stray etcd process could write to the directory while you move it; if found, `sudo kill <pid>` (or `kill -9`).

**Why stop all four containers, not just etcd?** Only etcd touches `/var/lib/etcd/`, so strictly only etcd must die. Stopping all four leaves each node clean and identical, with no orphaned apiserver trying to reconnect to a half-restored etcd, and everything starts fresh together later.

Sanity check before the destructive step (sample from cp03): `/var/lib/etcd` exists, root-owned, ~317 MB.

```bash
sudo ls -ld /var/lib/etcd
sudo du -sh /var/lib/etcd
```

## 19. Stage C: Preserve the current data directories

On **each** CP:

```bash
sudo mv /var/lib/etcd /var/lib/etcd.backup.$(date +%Y%m%d-%H%M%S)
sudo ls -ld /var/lib/etcd*
```

Expected: only the `.backup.<timestamp>` directory remains; `/var/lib/etcd` is gone.

```text
drwx------ 3 root root 4096 Oct  9 13:15 /var/lib/etcd.backup.20261009-153203
```

- **`mv`, never `rm`.** It is instant (same filesystem) and is your rollback: if the restore fails you move it back.
- It is a safety net, not a backup (same disk). Fine for the duration of the operation.
- Note the timestamp for cleanup later.

## 20. Stage D: Restore on each node

Run on **each** CP. `--initial-cluster` is **byte-identical** on all three; only `--name` and `--initial-advertise-peer-urls` differ.

| Node | Peer URL |
|---|---|
| `cp01.faekcorp.lab` | `https://10.70.21.6:2380` |
| `cp02.faekcorp.lab` | `https://10.70.31.209:2380` |
| `cp03.faekcorp.lab` | `https://10.70.41.250:2380` |

**cp01**

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp01.faekcorp.lab \
  --initial-cluster=cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380 \
  --initial-advertise-peer-urls=https://10.70.21.6:2380 \
  --data-dir=/var/lib/etcd
```

**cp02**

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp02.faekcorp.lab \
  --initial-cluster=cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380 \
  --initial-advertise-peer-urls=https://10.70.31.209:2380 \
  --data-dir=/var/lib/etcd
```

**cp03**

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp03.faekcorp.lab \
  --initial-cluster=cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380 \
  --initial-advertise-peer-urls=https://10.70.41.250:2380 \
  --data-dir=/var/lib/etcd
```

Using Path B (laptop hop)? Replace only the source path with `~/etcd-snapshot-cp01-20261006-223331.db`. Everything else is identical.

### Expected output

```text
info  snapshot/v3_snapshot.go  restoring snapshot  {"path": "...", "wal-dir": "/var/lib/etcd/member/wal", ...}
info  bbolt  Opening db file (/var/lib/etcd/member/snap/db) ...
info  schema/membership.go  Trimming membership information from the backend...
info  membership/cluster.go  added member  {"cluster-id": "7142046e27c4441c", "added-peer-id": "...", "added-peer-peer-urls": ["https://10.70.21.6:2380"], ...}
info  membership/cluster.go  added member  {"cluster-id": "7142046e27c4441c", "added-peer-id": "...", "added-peer-peer-urls": ["https://10.70.31.209:2380"], ...}
info  membership/cluster.go  added member  {"cluster-id": "7142046e27c4441c", "added-peer-id": "...", "added-peer-peer-urls": ["https://10.70.41.250:2380"], ...}
info  snapshot/v3_snapshot.go  restored snapshot  {"path": "...", "data-dir": "/var/lib/etcd", ...}
```

What to verify:

- **`cluster-id` is identical on all three nodes** (here `7142046e27c4441c`). This is the single most important check. A mismatch means they will not form one cluster; stop and re-run.
- All three peer URLs are listed.
- The final line says `restored snapshot`.
- `"local-member-id": "0"` during restore is normal; real member IDs are assigned on first start.

The restore takes seconds (12 MB, no network) and rebuilds the whole tree: `member/snap/db`, a fresh `.snap` checkpoint, and a fresh `member/wal/`.

## 21. Stage E: Fix permissions, start kubelet

On **each** CP:

```bash
sudo chmod 700 /var/lib/etcd
sudo chown -R root:root /var/lib/etcd
sudo ls -ld /var/lib/etcd          # expect: drwx------ ... root root ... /var/lib/etcd
sudo systemctl start kubelet
sudo crictl ps                     # control-plane containers should appear within seconds
```

`etcdutl` usually sets these correctly already; running them is belt-and-braces. etcd runs as root and expects `700 root:root`; wrong permissions make it fail with a permission error.

## 22. Stage F: Watch quorum reform

From a host with `kubectl` (bastion or a CP):

```bash
kubectl get pods -n kube-system -l component=etcd -w
```

```text
etcd-cp01.faekcorp.lab   0/1   ContainerCreating   0   5s
...
etcd-cp01.faekcorp.lab   1/1   Running   0   35s
etcd-cp02.faekcorp.lab   1/1   Running   0   33s
etcd-cp03.faekcorp.lab   1/1   Running   0   32s
```

Ctrl+C once all three are `1/1 Running`. Expect 30 to 90 seconds; up to a few minutes is still normal (leader election, WAL replay). The API may briefly refuse connections while the apiserver rediscovers etcd.

If after ~5 minutes only 1 or 2 are up:

```bash
kubectl logs -n kube-system etcd-cp01.faekcorp.lab --tail=50
sudo crictl logs <etcd-container-id> --tail=50      # if the API is down
```

Common stall causes: peers can't reach each other (firewall on 2380), mismatched `--initial-cluster`, different cluster IDs, a stray etcd process, wrong permissions.

---

# Part 5: Validation

## 23. etcd cluster health

```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health -w table
```

Expected: 3 members `started`, all endpoints `true`. **Member IDs are new** (e.g. `7296fe26dd7ee1e2` replaced `ee6b7ea6225a4e3e`): restore builds a new cluster identity. The member **names** stay the same.

## 24. Proving point-in-time correctness

**24.1 The canary test (strongest proof):**

```bash
kubectl get ns sim-backup        # MUST be NotFound
kubectl get ns after-snapshot    # MUST be NotFound
kubectl get ns                   # must show the 5 originals, not 7
```

**24.2 The revision test (numerical proof):**

```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

`REVISION` should be only slightly above the snapshot's `214775` (plus writes since restart), far below the pre-restore value. *(We did not capture this output in the drill; run it to complete the proof.)*

**24.3 Originals intact:**

```bash
kubectl get deploy -n default          # nginx-deploy 2/2
kubectl get deploy -n kube-system      # coredns 2/2, cilium-operator 1/1 (may take minutes, see section 26)
kubectl get ds -n kube-system          # cilium 5/5, cilium-envoy 5/5
kubectl get ns cilium-secrets

kubectl get all -A -o wide > ~/post-restore-cluster-state.txt
diff ~/pre-restore-cluster-state.txt ~/post-restore-cluster-state.txt
```

The diff will show the canaries gone plus unrelated churn (ages, restart counts).

**Result in our lab:**

| Check | Result |
|---|---|
| etcd, apiserver, controller-manager, scheduler on all 3 CPs | Running |
| All 5 nodes | Ready |
| `sim-backup`, `after-snapshot` | **NotFound (gone)** |
| `default`, `kube-system`, `cilium-secrets`, `nginx-deploy`, `coredns`, Cilium DaemonSets | Present |
| `cilium-operator` | Temporary CrashLoopBackOff, then recovered (section 26) |
| Worker containers from deleted namespaces | Still running (section 27) |

## 25. Workload and scheduling tests

```bash
kubectl run test-restore --image=nginx:latest --restart=Never
kubectl get pod test-restore
kubectl delete pod test-restore
```

If it reaches `Running 1/1`, then the scheduler works, kubelet can pull and run images, the CNI (Cilium) is functional, and the apiserver accepts writes.

> **Warning from our drill:** a plain test Deployment initially stayed `Pending` even though all nodes were `Ready` and the scheduler had assigned the Pods. See [section 27](#27-troubleshooting-case-stale-containers-and-pending-pods-on-workers) before concluding the restore is complete.

---

# Part 6: After the Restore: Things That Go Wrong

## 26. `cilium-operator` CrashLoopBackOff (Lease-based controllers)

**What we observed.** A few minutes after the restore:

```text
kube-system  pod/cilium-operator-56dcd959f4-4vdpc  0/1  CrashLoopBackOff  25 (2m39s ago)  3d22h
kube-system  deployment.apps/cilium-operator       0/1  1  0  5d17h
```

**What happened next.** Within minutes the same pod stabilized:

```text
kube-system  pod/cilium-operator-56dcd959f4-4vdpc  1/1  Running  26 (8m ago)  3d22h
kube-system  deployment.apps/cilium-operator       1/1  1  1  5d17h
```

The `26 (8m ago)` is the historical restart count; the crash loop ended.

**Likely explanation (plausible, not proven from logs).** `cilium-operator` uses a leader-election **Lease** stored in etcd. After the restore, the stored Lease is stale (its `renewTime` is from before the snapshot) and the operator's in-memory state disagrees with it for a short time. Once it re-acquires or renews the Lease, it settles. We did not capture the operator logs, so treat the Lease theory as the working explanation.

**What to do (default posture: wait and observe, do not reflexively delete pods):**

| Observation | Action |
|---|---|
| CrashLoop for < 5 minutes | Wait; it should settle |
| Persists > 10 minutes | `kubectl logs -n kube-system deploy/cilium-operator --tail=50` |
| Log shows a permanent error (not Lease contention) | Investigate that error; may be unrelated to the restore |
| Stuck after 15+ minutes | `kubectl rollout restart deployment -n kube-system cilium-operator` (or delete the pod) |

**Generalizes to** any controller using `coordination.k8s.io` Leases (Cilium, cert-manager, cloud controllers, leader-elected components). Most self-heal; escalate only if it persists beyond 10 to 15 minutes.

**Object ages vs container ages.** After restore, `kubectl get pods` shows old object ages (restored from the snapshot) but fresh container start times and restart counts. This is expected.

## 27. Troubleshooting case: stale containers and Pending Pods on workers

> This section is written as an honest incident record: what we **observed**, what we **suspected**, what we **changed**, and what we **proved**. Restarting kubelet helped us progress, but our evidence does **not** prove that restarting kubelet on every worker is always necessary after an etcd restore. Treat it as *the solution observed in this experiment*, not a universal Kubernetes requirement.

### 27.1 Situation

Five nodes: `cp01`/`cp02`/`cp03` (control planes), `worker1`/`worker2` (workers). kubeadm, containerd, Cilium.

After the restore the API looked right. On the bastion:

```bash
kubectl get ns
```

```text
NAME              STATUS   AGE
cilium-secrets    Active   5d18h
default           Active   5d23h
kube-node-lease   Active   5d23h
kube-public       Active   5d23h
kube-system       Active   5d23h
```

`after-snapshot` and `sim-backup` were gone, as expected. But the workers disagreed.

### 27.2 Problem one: containers from deleted namespaces still running

On worker1:

```bash
sudo crictl ps
```

```text
NAME   POD                                      NAMESPACE
nginx  after-snapshot-deploy-798d66d75b-xscvq   after-snapshot
nginx  sim-deploy-7d949546-f75cj                sim-backup
nginx  nginx-deploy-8b9dbd8c9-tjwgb             default
```

Worker2 showed the same pattern (`after-snapshot-deploy-...-rvr4g`, `sim-deploy-...-9kcs5`, `nginx-deploy-...-brsgn`).

**The contradiction:** Kubernetes no longer listed those namespaces, yet containerd still ran containers for Pods in them.

**Why this is possible: etcd and containerd hold different kinds of state.**

| Component | Responsibility |
|---|---|
| etcd | Stores Kubernetes API objects (cluster state) |
| kube-apiserver | Exposes the API backed by the restored state |
| kubelet | Reconciles the Pods assigned to a node with the desired state |
| containerd | Manages local container and Pod sandbox state |

Restoring etcd does not run a cleanup against containerd on every worker. It rewrites cluster state; it does not erase containers already running on a worker. Workers' kubelets were also left running during our restore (we stopped kubelet only on the control planes), which is a plausible contributor, but we did not prove it.

**First finding:** restoring etcd and cleaning up workers' runtime state are separate operations.

### 27.3 Problem two: a new Deployment stayed Pending

To test the restored cluster:

```bash
kubectl create ns after-restore
kubectl create deploy after-restore-deploy --image=nginx --replicas=2 -n after-restore
```

The Deployment was created, but its Pods stayed `Pending`:

```text
NAME                                   READY   STATUS
after-restore-deploy-759c7fb75b-fvckk  0/1     Pending
after-restore-deploy-759c7fb75b-l62j6  0/1     Pending

deployment.apps/after-restore-deploy   0/2   2   0
```

Desired 2, updated 2, available 0. The API server accepted the Deployment and the controller created the ReplicaSet and Pods; the Pods just never became operational. `kubectl get nodes` showed all five nodes `Ready`, which made a general node outage unlikely but did not prove every node component was healthy (a node can be `Ready` while a particular container startup is failing).

### 27.4 Checks performed (staged, no reflexive restarts)

**Check 1: Inspect the Pod.**

```bash
kubectl describe pod -n after-restore after-restore-deploy-759c7fb75b-fvckk
```

```text
Status: Pending
Conditions: PodScheduled  True
Events: Normal  Scheduled  Successfully assigned after-restore/after-restore-deploy-759c7fb75b-fvckk to worker1.faekcorp.lab
```

The scheduler had assigned the Pod (the other to worker2), so **the scheduler was not the obstacle**. The failure was later in Pod startup. However, the output did not show a `FailedCreatePodSandBox`, CNI error, or containerd error, so the exact cause was not identified.

**Check 2: Inspect the Deployment.**

```bash
kubectl describe deploy after-restore-deploy -n after-restore
```

```text
Replicas: 2 desired | 2 updated | 2 total | 0 available | 2 unavailable
Events: Scaled up replica set after-restore-deploy-759c7fb75b from 0 to 2
```

The Deployment controller was working; the problem was further down the path.

**Check 3: Inspect containerd on the workers.**

```bash
sudo crictl ps
```

Containerd was running and managing CoreDNS, the old NGINX workloads, Cilium agent and Cilium Envoy. This showed containerd was alive, not that new containers or sandboxes could be created.

**Check 4: Restart kubelet on worker2 (the key intervention).**

```bash
sudo systemctl restart kubelet.service
sudo crictl ps
```

Immediately after the restart, the `after-snapshot` NGINX container disappeared from the running list; `sim-backup` was still there. Later, that container and CoreDNS were also absent, while the Cilium operator, agent, Envoy and the default NGINX container kept running. This was consistent with kubelet reconciling local runtime state after restart.

**Important qualification:** `crictl ps` shows only *running* containers. A container vanishing from it may have exited or been removed. We did not capture `crictl ps -a` or `crictl pods -a` at that moment, so we cannot say which cleanup actually occurred.

### 27.5 Solution observed

After the worker-side investigation we created the Deployment again:

```bash
kubectl create deploy after-restore-deploy --image=nginx --replicas=2 -n after-restore
```

This time both Pods started:

```text
NAME                                   READY   STATUS    RESTARTS
after-restore-deploy-759c7fb75b-qr6zg  1/1     Running   0
after-restore-deploy-759c7fb75b-xf4xl  1/1     Running   0

deployment.apps/after-restore-deploy   2/2   2   2
```

**Established:**

1. The restored cluster accepted a new namespace and Deployment.
2. The scheduler assigned Pods to workers.
3. Restarting kubelet on worker2 was followed by stale containers leaving the running list.
4. A subsequent two-replica NGINX Deployment reached `2/2`.

**Not established:**

- That the kubelet restart alone fixed the *original* Pending Pods (the success came from a **newly created** Deployment).
- The precise reason the original Pods stayed Pending (not enough runtime logs were captured).

So describe it as: *restarting kubelet on the affected worker, then successfully retesting with a new Deployment*, not as a conclusively isolated root cause.

### 27.6 Why restarting kubelet can help

```text
          ETCD
            │
            ▼
     Kubernetes API
            │
            ▼
         KUBELET  ── reconciliation ──►  CONTAINERD  ──►  local containers/sandboxes
```

If a container is running for a Pod that no longer exists in the restored API state, kubelet is responsible for reconciling it. Restarting kubelet forces fresh initialization and reconciliation, which can clear certain stale-state situations. But kubelet reconciles continuously, so a restart should **not** be assumed mandatory after every restore. Verify convergence, restart kubelet on an *affected* worker when evidence supports it, then verify recovery.

Do not confuse this with restarting **containerd**. They are different services; we restarted **kubelet**, not containerd.

### 27.7 Recommended verification procedure (use this after every restore)

**Phase A: Verify restored API state (bastion):**

```bash
kubectl get nodes
kubectl get ns
kubectl get deployments -A
kubectl get pods -A -o wide
```

Confirm the objects match the expected snapshot state.

**Phase B: Inspect worker runtime state (each worker):**

```bash
sudo systemctl status kubelet --no-pager
sudo systemctl status containerd --no-pager
sudo crictl ps -a
sudo crictl pods -a
```

Use `-a`: running-only output cannot distinguish running, exited and absent objects.

**Phase C: Investigate reconciliation problems:**

```bash
sudo journalctl -u kubelet --since "30 minutes ago" --no-pager
```

If there is evidence kubelet is not reconciling, restart it on the affected worker **one worker at a time** (never all simultaneously):

```bash
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

Recheck runtime state and logs after each restart.

**Phase D: Verify workload recovery (bastion):**

```bash
kubectl get nodes
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.metadata.creationTimestamp

kubectl create namespace restore-validation
kubectl create deployment nginx-test --image=nginx --replicas=2 -n restore-validation
kubectl rollout status deployment/nginx-test -n restore-validation
kubectl get pods -n restore-validation -o wide
```

Expected: two Pods `Running`, `2/2` available. Clean up if desired:

```bash
kubectl delete namespace restore-validation
```

### 27.8 Incident summary

| Item | Finding |
|---|---|
| Incident | Stale containers remained on workers after the etcd restore; a new NGINX Deployment initially stayed Pending |
| Initial symptom | Namespaces vanished from the API but containers for their old Pods kept running |
| First checks | `kubectl get ns`, `get nodes`, `describe pod`, `describe deploy`, `crictl ps` |
| Scheduling | Both Pods were assigned to workers successfully |
| Intervention | Restarted kubelet on worker2 |
| Observed change | Some old containers disappeared from `crictl ps` |
| Successful validation | A newly created two-replica NGINX Deployment reached `2/2` |
| Root cause status | Kubelet reconciliation is plausible; the precise cause of the original Pending Pods was **not** conclusively established |
| Operational lesson | After restoring etcd, verify **both** the Kubernetes API state **and** worker runtime state; do not assume the restore cleans up local containers |

**Next step for a stronger case:** in a controlled reproduction, capture `crictl ps -a`, `crictl pods -a`, and the kubelet logs (`journalctl -u kubelet`) while the Pods are Pending, and `kubectl describe pod` events for `FailedCreatePodSandBox` / CNI errors.

---

# Part 7: Cleanup and Production

## 28. Cleanup: what to keep and when to delete

| Item | Keep for | Why |
|---|---|---|
| `~/s3-etcd-backups/<snapshot>.db` (each CP) | Until the next successful drill | Source of truth for this restore |
| `~/etcd-snapshot-*.db` (each CP, Path B) | Until the next successful drill | Same, if using the laptop hop |
| S3 object `s3://.../etcd-snapshots/<snapshot>.db` | Per retention policy | The durable backup |
| `/var/lib/etcd.backup.<timestamp>` (each CP) | **At least 24 h** (24 to 48 h in production) | Rollback safety if issues surface |
| `~/etcd.yaml.before-restore`, `~/etcd-certs.before-restore.txt` | At least 24 h | Compare if anything looks odd |
| `~/pre-restore-cluster-state.txt`, `~/post-restore-cluster-state.txt` | At least 24 h | Diff resource state |

Delete immediately: any `/tmp/snapshot.db` you staged.

Delete after 24 to 48 h of confirmed stable operation:

```bash
sudo rm -rf /var/lib/etcd.backup.*
```

Never remove the backup directories before verifying the restore. Also remove any test namespaces you created (`after-restore`, `restore-validation`).

## 29. Production considerations

### RTO and RPO

| Phase | Duration |
|---|---|
| Pre-flight + canary creation | 5 to 10 min |
| Stage B (stop kubelet + force-stop containers) | 2 to 4 min |
| Stage C (mv data dirs) | < 1 min |
| Stage D (`etcdutl` restore, 3 nodes) | 1 to 2 min |
| Stage E (perms + start kubelet) | < 1 min |
| Stage F (quorum reform) | 1 to 3 min |
| Validation | 5 to 10 min |
| Lease-based controllers + worker reconciliation | 3 to 15+ min (background) |
| **Total RTO, API usable** | **~15 to 25 min** |
| **Total RTO, fully settled** | **~30 to 40 min** |

**RPO = snapshot frequency.** Hourly snapshots mean up to 1 hour of data loss; daily means up to 24 hours.

### Do not automate the restore

Restore is high-stakes and keeps a human in the loop by design: you must choose *which* snapshot, verify the cluster's current state first, and validate afterward. A botched restore can be worse than the original problem.

| You CAN automate | Keep manual |
|---|---|
| Backups (systemd timer, per the companion doc) | The destructive `mv /var/lib/etcd` |
| Snapshot integrity checks and alerting | `etcdutl snapshot restore` on each node |
| Pre-flight checks (snapshot availability/integrity) | The go/no-go decision at each stage |
| Periodic restore drills on an isolated test cluster | |

### Other guidance

- **Do not run backups as an in-cluster CronJob.** If the cluster is broken, the CronJob can't run. Use a systemd timer on a control plane or an external scheduler.
- **Encrypt snapshots at rest.** They contain every Secret in the cluster. Use S3 server-side encryption or, e.g., `openssl enc -aes-256-cbc`.
- **Least privilege.** Attach an IAM role to the control-plane instances; avoid long-lived static keys on nodes.
- **Run restore drills regularly** (monthly or quarterly) on a non-production cluster with the same topology. Time them, record the RTO, fix surprises before they hit production. A backup that has never been restored is only a hope.
- **Announce the maintenance window.** The API is down while you work.
- **Expect divergence.** Anything outside etcd (PV data, cloud resources, worker runtime state, controller in-memory state) is not rewound and may need reconciling.

---

# Part 8: Reference

## 30. Complete command reference

**Snapshot verification**

| Command | Purpose |
|---|---|
| `etcdutl snapshot status <path> -w table` | Verify snapshot integrity (offline) |
| `sha256sum <path>` | Cross-check file integrity |
| `ls -lh <path>` | Confirm size |

**etcd inspection (while running)**

| Command | Purpose |
|---|---|
| `etcdctl member list -w table` | Cluster membership |
| `etcdctl endpoint health -w table` | Per-endpoint health |
| `etcdctl endpoint status -w table` | Revision, DB size, leader |

(All with `--endpoints=https://127.0.0.1:2379 --cacert=... --cert=... --key=...` and `sudo`.)

**Restore workflow**

| Stage | Command |
|---|---|
| A | `sudo cp /etc/kubernetes/manifests/etcd.yaml ~/etcd.yaml.before-restore` |
| B | `sudo systemctl stop kubelet` |
| B | `sudo crictl ps \| grep -E 'etcd\|apiserver\|controller-manager\|scheduler'` |
| B | `sudo crictl stop <container-id>` / `sudo crictl rm -f <container-id>` |
| B | `sudo ss -tlnp \| grep -E '2379\|2380'` (must be free) |
| C | `sudo mv /var/lib/etcd /var/lib/etcd.backup.$(date +%Y%m%d-%H%M%S)` |
| D | `sudo etcdutl snapshot restore <path> --name=... --initial-cluster=... --initial-advertise-peer-urls=... --data-dir=/var/lib/etcd` |
| E | `sudo chmod 700 /var/lib/etcd && sudo chown -R root:root /var/lib/etcd` |
| E | `sudo systemctl start kubelet` |
| F | `kubectl get pods -n kube-system -l component=etcd -w` |

**AWS CLI**

| Command | Purpose |
|---|---|
| `aws sts get-caller-identity` | Verify IAM role |
| `aws s3 ls s3://<bucket>/etcd-snapshots/` | List snapshots |
| `aws s3 cp s3://<bucket>/etcd-snapshots/<file> <dir>/` | Download (keeps filename) |

**SSH/SCP via bastion**

| Command | Purpose |
|---|---|
| `scp -i <cp-key> -o ProxyCommand="ssh -i <bastion-key> -W %h:%p ubuntu@<bastion>" <local> <dest>` | Copy through the bastion tunnel |
| `-o IdentitiesOnly=yes` | Force use of the specified key only |

**Worker diagnostics**

| Command | Purpose |
|---|---|
| `sudo crictl ps -a` / `sudo crictl pods -a` | All containers/sandboxes, incl. exited |
| `sudo systemctl status kubelet containerd --no-pager` | Service state |
| `sudo journalctl -u kubelet --since "30 minutes ago" --no-pager` | Kubelet logs |
| `sudo systemctl restart kubelet` | Force reconciliation (one worker at a time) |

## 31. Key paths

| Path | What it is |
|---|---|
| `/etc/kubernetes/manifests/etcd.yaml` | etcd static pod manifest |
| `/etc/kubernetes/pki/etcd/ca.crt` | etcd cluster CA cert |
| `/etc/kubernetes/pki/etcd/server.crt`, `server.key` | etcd server cert and key |
| `/etc/kubernetes/pki/etcd/healthcheck-client.crt`, `.key` | Read-only health-check client cert/key |
| `/var/lib/etcd/` | Live etcd data directory |
| `/var/lib/etcd/member/snap/db` | Live bbolt database |
| `/var/lib/etcd/member/wal/` | Write-ahead log |
| `/var/lib/etcd.backup.<ts>/` | Pre-restore data (rollback safety) |
| `~/s3-etcd-backups/` | Snapshots downloaded from S3, ready to restore |
| `~/etcd-snapshot-*.db` | Snapshots created at backup time, or copied via laptop hop |
| `/usr/local/bin/etcdctl`, `etcdutl` | etcd client and offline utility |
| `~/.aws/credentials` | Should **not** exist on CPs (use an IAM role) |

## 32. Troubleshooting table

| Symptom | Cause | Fix |
|---|---|---|
| `scp` cp01 to cp02: `Permission denied (publickey)` | CPs can't SSH each other; only bastion routes to all | Path B: laptop + `ProxyCommand` via bastion |
| Containers still `Running` after `systemctl stop kubelet` | Static-pod shutdown grace period / unclean stop | `crictl stop <container-id>`, then `crictl rm -f <id>` |
| `crictl stop <pod-name>` does nothing | crictl needs the container ID | Use the ID from `crictl ps` |
| `unrecognized environment variable ETCDCTL_API=3` | Deprecated in 3.6 | Drop the variable (harmless) |
| `etcdctl snapshot restore` missing/deprecated | Removed in 3.6 | Use `etcdutl snapshot restore` |
| Restore refuses to run / conflicts | `/var/lib/etcd` already exists | `mv` it aside first (Stage C) |
| etcd fails to start, permission error | Wrong ownership/mode | `chmod 700` + `chown -R root:root /var/lib/etcd` |
| No quorum / split brain | Not all nodes restored from the same snapshot, or `--initial-cluster` differs | Restore all three, same snapshot, identical `--initial-cluster`; compare `cluster-id` in logs |
| etcd pod won't start | Stray etcd process / port in use | `sudo ss -tlnp \| grep 2379`, kill the process |
| Cert errors | Missing files in `/etc/kubernetes/pki/etcd/` | `sudo ls -l /etc/kubernetes/pki/etcd/` |
| `etcdctl` can't connect | Wrong endpoint or no sudo | Use `--endpoints=https://127.0.0.1:2379` with `sudo` |
| API `connection refused` right after restore | apiserver still rediscovering etcd | Wait 3 to 5 min; then check `kubectl logs ... kube-apiserver-cpXX` |
| Resources missing after restore | Wrong/old snapshot, or restore failed | Check snapshot REVISION/TOTAL KEYS; check etcd logs |
| `cilium-operator` CrashLoopBackOff | Stale Lease right after restore | Wait; escalate only if > 15 min (section 26) |
| Containers from deleted namespaces still on workers | etcd restore doesn't clean worker runtime | Verify with `crictl ps -a`; restart kubelet on the affected worker (section 27) |
| New Deployment stuck `Pending`, nodes `Ready`, scheduler assigned | Worker-side startup/reconciliation problem | Inspect kubelet logs, `crictl pods -a`; restart kubelet one worker at a time; retest |
| Control-plane pods with many restarts before the drill | Unrelated flapping | `kubectl describe pod` and logs before doing surgery |

## 33. One-page flow and quick sequence

```text
┌──────────────────────────────────────────────────────────────┐
│ Pre-flight │ Snapshot on all 3 CPs, same HASH; etcd healthy   │
├──────────────────────────────────────────────────────────────┤
│ Canaries   │ Create post-snapshot namespaces + deployments    │
├──────────────────────────────────────────────────────────────┤
│ Sourcing   │ A) aws s3 cp → ~/s3-etcd-backups/                │
│            │ B) scp via laptop → bastion → cp02/cp03          │
│            │ Verify HASH identical on all 3                   │
├──────────────────────────────────────────────────────────────┤
│ Stage A    │ Save current manifest (read-only safety)         │
│ Stage B    │ Stop kubelet; force-stop containers by ID;       │
│            │ ports 2379/2380 free                             │
│ Stage C    │ mv /var/lib/etcd → /var/lib/etcd.backup.<ts>     │
│ Stage D    │ etcdutl snapshot restore on each node            │
│            │ same --initial-cluster; cluster-id must match    │
│ Stage E    │ chmod 700, chown root, start kubelet             │
│ Stage F    │ Watch etcd pods reform quorum                    │
├──────────────────────────────────────────────────────────────┤
│ Validate   │ etcd healthy; canaries NotFound; originals OK    │
├──────────────────────────────────────────────────────────────┤
│ Settle     │ Lease controllers may CrashLoop briefly (wait)   │
│            │ Check workers: crictl ps -a, kubelet logs;       │
│            │ restart kubelet on affected worker if needed     │
│            │ Retest with a fresh Deployment                   │
├──────────────────────────────────────────────────────────────┤
│ Cleanup    │ Keep .backup dirs ≥ 24 h, then remove            │
└──────────────────────────────────────────────────────────────┘
```

Quick command sequence (placeholders in `<>`):

```bash
# ── PRE-FLIGHT / CANARIES (bastion) ───────────────────────────────
kubectl get all -A -o wide > ~/pre-restore-cluster-state.txt
kubectl create ns sim-backup && kubectl create ns after-snapshot
kubectl create deploy sim-deploy            --image=nginx --replicas 2 -n sim-backup
kubectl create deploy after-snapshot-deploy --image=nginx --replicas 2 -n after-snapshot

# ── SOURCING (ALL CPs, cluster still healthy) ─────────────────────
mkdir -p ~/s3-etcd-backups
aws s3 cp s3://<BUCKET>/etcd-snapshots/<SNAPSHOT>.db ~/s3-etcd-backups/
etcdutl snapshot status ~/s3-etcd-backups/<SNAPSHOT>.db -w table    # HASH must match everywhere

# ── STAGE A/B (ALL CPs) ───────────────────────────────────────────
sudo cp /etc/kubernetes/manifests/etcd.yaml ~/etcd.yaml.before-restore
sudo systemctl stop kubelet
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler'
sudo crictl stop <ids...>                 # force: sudo crictl rm -f <id>
sudo ss -tlnp | grep -E '2379|2380' || echo "ports free"

# ── STAGE C (ALL CPs) ─────────────────────────────────────────────
sudo mv /var/lib/etcd /var/lib/etcd.backup.$(date +%Y%m%d-%H%M%S)

# ── STAGE D (EACH CP: node-specific --name / --initial-advertise-peer-urls)
sudo etcdutl snapshot restore ~/s3-etcd-backups/<SNAPSHOT>.db \
  --name=<NODE_NAME> \
  --initial-cluster=<cp01>=https://<IP1>:2380,<cp02>=https://<IP2>:2380,<cp03>=https://<IP3>:2380 \
  --initial-advertise-peer-urls=https://<THIS_NODE_IP>:2380 \
  --data-dir=/var/lib/etcd

# ── STAGE E (ALL CPs) ─────────────────────────────────────────────
sudo chmod 700 /var/lib/etcd && sudo chown -R root:root /var/lib/etcd
sudo systemctl start kubelet

# ── STAGE F / VALIDATE (bastion) ──────────────────────────────────
kubectl get pods -n kube-system -l component=etcd -w
kubectl get nodes
kubectl get ns sim-backup        # must be NotFound
kubectl get ns after-snapshot    # must be NotFound

# ── WORKER CHECK (each worker, one at a time) ─────────────────────
sudo crictl ps -a && sudo crictl pods -a
sudo journalctl -u kubelet --since "30 minutes ago" --no-pager
sudo systemctl restart kubelet           # only if reconciliation is not happening

# ── FINAL TEST (bastion) ──────────────────────────────────────────
kubectl create ns restore-validation
kubectl create deployment nginx-test --image=nginx --replicas=2 -n restore-validation
kubectl rollout status deployment/nginx-test -n restore-validation
```

## 34. Glossary

| Term | Meaning |
|---|---|
| **etcd** | Distributed key-value store holding all Kubernetes cluster state |
| **Raft** | Consensus protocol etcd uses to replicate writes across members |
| **Quorum** | Majority of members (2 of 3) needed to commit writes |
| **Static pod** | Pod run directly by kubelet from a manifest in `/etc/kubernetes/manifests` |
| **Snapshot** | Portable logical dump of etcd's key-value state |
| **Cluster ID / Member ID** | Identities baked in when a cluster/member is created; restore generates new ones |
| **Lease** | Kubernetes object (`coordination.k8s.io`) used for leader election and heartbeats |
| **kubelet** | Node agent that reconciles local Pods/containers with the API's desired state |
| **containerd / crictl** | Container runtime / its CLI for inspecting containers and sandboxes |
| **Bastion** | Jump host through which the private control planes are reached |
| **RPO / RTO** | Recovery Point Objective (data-loss window) / Recovery Time Objective (downtime) |
