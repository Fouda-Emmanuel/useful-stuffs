# Kubernetes etcd Restore from Snapshot (S3 Bucket) — Complete Runbook

A hands-on, step-by-step guide for restoring a 3-member stacked etcd cluster on a kubeadm-based Kubernetes cluster from a snapshot, including point-in-time verification and post-restore worker reconciliation.

Written from real experiments on a 3-control-plane HA cluster:
- **Kubernetes:** v1.35.8
- **etcd:** 3.6.6 (image `registry.k8s.io/etcd:3.6.6-0`)
- **Cluster name:** faekcorp-lab
- **Nodes:** cp01, cp02, cp03 (stacked etcd) + worker1, worker2
- **Load balancer:** HAProxy at `haproxy.faekcorp.lab:6443`
- **Bastion:** kubectl-only admin host (with laptop-hop `ProxyCommand` access)
- **S3 bucket:** `ha-cluster-s3-etcd-lab` (us-east-1)
- **Companion doc:** *Kubernetes etcd Backup & S3 Upload — Complete Runbook*

---

## Table of Contents

**Part 1 — Why Restore Is Different from Backup**
1. The mental model
2. Restore is a build, not a copy
3. Why all members must be restored from the same snapshot
4. What survives, what disappears

**Part 2 — The Architecture We're Working With**
5. Cluster topology
6. Peer URLs and the `--initial-cluster` contract
7. Where the tools and data live

**Part 3 — Understanding `/var/lib/etcd/` After Restore**
8. The three different "snapshots"
9. What `etcdutl snapshot restore` actually builds
10. Why the directory must be emptied first

**Part 4 — Where the Snapshot Comes From**
11. Local file vs S3 vs laptop-hop
12. Path A — Download from S3
13. Path B — Copy via admin laptop through the bastion
14. Verifying the snapshot on all three CPs

**Part 5 — The Restore Procedure**
15. Stage A — Save current manifest (read-only safety)
16. Stage B — Stop kubelet on all control planes
17. Force-stopping static pod containers (the crictl-by-ID gotcha)
18. Stage C — Preserve the current data directories
19. Stage D — Restore on each node
20. Stage E — Fix permissions, restart kubelet
21. Stage F — Watch quorum reform

**Part 6 — Validating the Restore**
22. etcd cluster health
23. Proving point-in-time correctness (canary + revision)
24. Workload and scheduling tests
25. The cilium-operator CrashLoop that self-heals
26. Worker nodes after the restore — stale containers and Pending Pods

**Part 7 — Post-Restore Cleanup**
27. What to keep, what to delete, when

**Part 8 — Production Considerations**
28. RTO and RPO
29. Automating restores (don't)
30. Regular restore drills

**Part 9 — Reference**
31. Complete command reference
32. Key paths
33. Troubleshooting
34. Summary — the whole flow at a glance

---

# Part 1 — Why Restore Is Different from Backup

## 1. The Mental Model

Backup = read. Restore = build.

`etcdctl snapshot save` produces a **portable logical dump** — every key, every value, every revision number, the cluster membership, the cluster ID at snapshot time. It is **not** a copy of `/var/lib/etcd/`.

`etcdutl snapshot restore` reads that dump and **builds a fresh etcd data directory** on disk that looks like a just-bootstrapped member, pre-loaded with the snapshot's data. It also:

- Assigns a **new cluster ID**
- Assigns **new member IDs**
- Writes a **new membership list** (from the `--initial-cluster` flag you supply)

The snapshot is the brain. The `--data-dir` is where the new body gets built. `--name` and `--initial-advertise-peer-urls` tell the new body *who it is*.

## 2. Restore Is a Build, Not a Copy

```
etcdctl snapshot save:
    live cluster  ──►  portable logical dump (one file)

etcdutl snapshot restore:
    portable dump  ──►  fresh etcd data directory
                        (member/snap/db + wal + metadata)
```

One is export. The other is import-and-bootstrap.

## 3. Why All Members Must Be Restored from the Same Snapshot

If you restore only cp01:

```
cp01 (restored):   "I'm in cluster XYZ, peers are A/B/C"
cp02 (untouched):  "I'm in cluster ABC, peers are A/B/C"
cp03 (untouched):  "I'm in cluster ABC, peers are A/B/C"
```

cp01 and cp02 refuse to talk — different cluster IDs. You end up with **split brain**: two clusters that both think they're real.

Raft replicates **log entries** between members of the *same cluster*. It cannot:
- Change a member's cluster ID
- Merge two clusters
- Accept a member with a different cluster ID

The only way to get all three members into one new cluster is to restore **all three from the same snapshot**, with **identical `--initial-cluster`**, and node-specific `--name` / `--initial-advertise-peer-urls`.

## 4. What Survives, What Disappears

The snapshot represents a **specific moment** in the cluster's history. After restore:

| Category | Behavior |
|----------|----------|
| Resources created **before** the snapshot | ✅ Survive |
| Resources created **after** the snapshot | ❌ Vanish |
| Modifications made after the snapshot | ❌ Reverted |
| Deletions made after the snapshot | ✅ Restored |

This is the whole point: restore is **atomic rollback to the snapshot's exact state**.

---

# Part 2 — The Architecture We're Working With

## 5. Cluster Topology

```
                    ┌──────────────────┐
                    │     Bastion      │
                    │  kubectl only    │
                    └────────┬─────────┘
                             │ :6443
                             ▼
                    ┌──────────────────┐
                    │  HAProxy + DNS   │
                    │  haproxy.lab:6443│
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          ┌───────┐      ┌───────┐      ┌───────┐
          │ cp01  │      │ cp02  │      │ cp03  │
          │ etcd  │◄────►│ etcd  │◄────►│ etcd  │
          │ apiser│      │ apiser│      │ apiser│
          │ CM    │      │ CM    │      │ CM    │
          │ sched │      │ sched │      │ sched │
          └───────┘      └───────┘      └───────┘

          ┌───────────┐         ┌───────────┐
          │ worker1   │         │ worker2   │
          │ kubelet   │         │ kubelet   │
          │ kube-proxy│         │ kube-proxy│
          │ cilium    │         │ cilium    │
          └───────────┘         └───────────┘
```

Key facts:

- **3 control-plane nodes** with **stacked etcd**
- **2 worker nodes** — no etcd, no control-plane components
- etcd runs as a **static pod** (managed by kubelet)
- etcd uses **mTLS** — client certs required for admin operations
- Certs live under `/etc/kubernetes/pki/etcd/`
- Data lives under `/var/lib/etcd/`
- **CPs cannot SSH to each other directly** — admin access is via the bastion, or from the admin laptop through a `ProxyCommand` tunnel
- **Workers are not touched by the restore procedure itself** — they have no etcd, and their kubelet keeps running through the whole procedure (only the API is unavailable to them). But their local container runtime state can drift from the restored API state — see Section 26.

## 6. Peer URLs and the `--initial-cluster` Contract

Our actual peer URLs:

| Node | Peer URL |
|------|----------|
| cp01.faekcorp.lab | `https://10.70.21.6:2380` |
| cp02.faekcorp.lab | `https://10.70.31.209:2380` |
| cp03.faekcorp.lab | `https://10.70.41.250:2380` |

The shared `--initial-cluster` string (identical on all three restore commands):

```
cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380
```

**Every node must pass the identical string.** That's how they recognize each other as members of the same new cluster.

> **Common confusion:** during normal cluster life, each node's `/etc/kubernetes/manifests/etcd.yaml` may show a different (stale) `--initial-cluster` — because that flag is only used on **first bootstrap** of an empty data directory. Once a data directory is populated, etcd reads membership from it. But during restore, the data directory is empty (we moved the old one away), so the flag we pass to `etcdutl snapshot restore` **is** consulted. It must be complete and identical everywhere.

## 7. Where the Tools and Data Live

| Component | Location | Purpose |
|-----------|----------|---------|
| `kubectl` | Bastion | Kubernetes API client |
| `etcdctl` | cp01 (and cp02/cp03 for validation) | etcd client |
| `etcdutl` | cp01 (and cp02/cp03) | etcd offline utility |
| `aws` CLI | cp01, cp02, cp03 | S3 download |
| etcd certs | `/etc/kubernetes/pki/etcd/` on each CP | mTLS |
| etcd live data | `/var/lib/etcd/` on each CP | etcd storage |
| Pre-restore backup | `/var/lib/etcd.backup.<timestamp>/` | Rollback safety |
| Downloaded snapshot | `~/s3-etcd-backups/*.db` (Path A) or `~/etcd-snapshot-*.db` (Path B) | Restore source |
| Worker runtime state | containerd on each worker | Persists across restore; may need reconciliation |

---

# Part 3 — Understanding `/var/lib/etcd/` After Restore

## 8. The Three Different "Snapshots"

| Name | Path | What it is | Used for restore? |
|------|------|-----------|-------------------|
| Internal checkpoint | `/var/lib/etcd/member/snap/*.snap` | etcd's own Raft metadata checkpoints | ❌ |
| Live database | `/var/lib/etcd/member/snap/db` | The live bbolt DB file | ❌ |
| Backup snapshot | `etcdctl snapshot save <path>` | Portable logical dump | ✅ This is what we restore |

## 9. What `etcdutl snapshot restore` Actually Builds

When you run:

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp01.faekcorp.lab \
  --initial-cluster=... \
  --initial-advertise-peer-urls=https://10.70.21.6:2380 \
  --data-dir=/var/lib/etcd
```

`etcdutl`:

1. Reads the portable snapshot
2. **Builds a brand-new directory tree** at `/var/lib/etcd`
3. Populates `member/snap/db` with the snapshot's data
4. Creates a fresh `member/wal/`
5. Writes the new cluster ID and membership (from the flags) into the new directory

Result: `/var/lib/etcd/` looks like a freshly bootstrapped member that happens to already contain the snapshot's data.

## 10. Why the Directory Must Be Emptied First

`etcdutl snapshot restore` **will not write into a populated directory**. It expects to build fresh.

That's why Stage C moves the current `/var/lib/etcd` out of the way **before** restore runs. That move is also our rollback safety net.

---

# Part 4 — Where the Snapshot Comes From

## 11. Local File vs S3 vs Laptop-Hop

| Source | When to use | Pros | Cons |
|--------|-------------|------|------|
| **S3 (Path A)** | Real disaster recovery; CPs have IAM role + internet | Durable, off-site, production path | Requires IAM role + AWS CLI + network on each CP |
| **Laptop hop (Path B)** | CPs can't reach S3, or can't SSH each other | Works in hardened topologies | Requires laptop + bastion + keys |

In our lab we documented both. Path A is the recommended production path. Path B was the working path during the drill because CPs can't SSH each other directly.

## 12. Path A — Download from S3

On **each CP** (cp01, cp02, cp03):

**Command:**

```bash
# 1. Confirm IAM role provides credentials
aws sts get-caller-identity

# 2. Confirm the bucket is reachable
aws s3 ls s3://ha-cluster-s3-etcd-lab/etcd-snapshots/

# 3. Create the local staging directory
mkdir -p ~/s3-etcd-backups

# 4. Download — keep the original dated filename
aws s3 cp s3://ha-cluster-s3-etcd-lab/etcd-snapshots/etcd-snapshot-cp01-20261006-223331.db \
  ~/s3-etcd-backups/

# 5. Verify
ls -lh ~/s3-etcd-backups/
etcdutl snapshot status ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db -w table
```

**Output (verify step):**

```
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| fc420e4c |   214775 |        387 |      12 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

**The HASH must be identical on all three nodes.** If not, stop and re-download.

## 13. Path B — Copy via Admin Laptop Through the Bastion

In the faekcorp lab, the control planes have **no direct SSH path to each other**. Attempting direct `scp` from cp01 fails.

**Command (on cp01):**

```bash
scp etcd-snapshot-cp01-20261006-223331.db ubuntu@10.70.1.6:~
```

**Output:**

```
The authenticity of host '10.70.1.6 (10.70.1.6)' can't be established.
ED25519 key fingerprint is SHA256:kWw3nWn/BfQsptQG8+1kVYsDuLYBdQVxTkvSYO5ingE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.70.1.6 (ED25519)' to the list of known hosts.
ubuntu@10.70.1.6: Permission denied (publickey).
scp: Connection closed
```

We worked around this by hopping through the admin laptop, using `ProxyCommand` to tunnel via the bastion. Each `scp` uses **two keys**: the control-plane keypair (destination auth) and the bastion keypair (jump host auth).

### Step 1 — Pull the snapshot from cp01 to the laptop

**Command (on the laptop):**

```bash
scp -o IdentitiesOnly=yes \
    -i ~/.ssh/lab-controlplane-keypair.pem \
    -o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem -o IdentitiesOnly=yes -W %h:%p ubuntu@100.61.170.135" \
    ubuntu@10.70.21.6:~/etcd-snapshot-cp01-20261006-223331.db \
    ~/
```

### Step 2 — Push to cp02 via the laptop → bastion → cp02 tunnel

**Command (on the laptop):**

```bash
scp -o IdentitiesOnly=yes \
    -i ~/.ssh/lab-controlplane-keypair.pem \
    -o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem -o IdentitiesOnly=yes -W %h:%p ubuntu@100.61.170.135" \
    ~/etcd-snapshot-cp01-20261006-223331.db \
    ubuntu@10.70.31.209:~/
```

**Output (first run only — host key prompt):**

```
The authenticity of host '10.70.31.209 (<no hostip for proxy command>)' can't be established.
ED25519 key fingerprint is SHA256:Pkm3WhzVQ6nlAXqLi1hZr0UYlt+aaMZP3YmG03MKKrw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.70.31.209 (<no hostip for proxy command>)' to the list of known hosts.
etcd-snapshot-cp01-20261006-223331.db    100%   12MB 448.3KB/s   00:26
```

> On subsequent runs (host key already in `known_hosts`), only the last line appears. The fingerprint prompt appears **once per destination host** the first time you connect.

### Step 3 — Push to cp03 the same way

**Command (on the laptop):**

```bash
scp -o IdentitiesOnly=yes \
    -i ~/.ssh/lab-controlplane-keypair.pem \
    -o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem -o IdentitiesOnly=yes -W %h:%p ubuntu@100.61.170.135" \
    ~/etcd-snapshot-cp01-20261006-223331.db \
    ubuntu@10.70.41.250:~/
```

**Output (first run only — host key prompt):**

```
The authenticity of host '10.70.41.250 (<no hostip for proxy command>)' can't be established.
ED25519 key fingerprint is SHA256:GtMubmt13hD+EJALjAcVKPptuvIyMLYg4CSLb7qKm6o.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.70.41.250 (<no hostip for proxy command>)' to the list of known hosts.
etcd-snapshot-cp01-20261006-223331.db    100%   12MB 427.2KB/s   00:28
```

### What each flag does

| Flag | Meaning |
|------|---------|
| `-i ~/.ssh/lab-controlplane-keypair.pem` | The key that authenticates to the destination CP (`ubuntu@10.70.31.209`) |
| `-o IdentitiesOnly=yes` | Use **only** the key specified with `-i`, ignoring ssh-agent |
| `-o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem ... -W %h:%p ubuntu@100.61.170.135"` | SSH connects to the bastion, then uses `-W %h:%p` to **forward raw TCP** to the destination |

### Why the laptop hop is necessary here

- cp01 and cp02/cp03 are on different network segments that cannot reach each other directly
- Only the bastion has routes to all three
- The admin laptop authenticates to the bastion (with `lab-bastion-keypair.pem`) and to each CP (with `lab-controlplane-keypair.pem`)

## 14. Verifying the Snapshot on All Three CPs

Regardless of which path you took.

**Command:**

```bash
# Path A staging location
ls -lh ~/s3-etcd-backups/
etcdutl snapshot status ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db -w table

# Path B staging location (home dir)
ls -lh ~/etcd-snapshot-cp01-20261006-223331.db
etcdutl snapshot status ~/etcd-snapshot-cp01-20261006-223331.db -w table
```

**Output:**

```
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| fc420e4c |   214775 |        387 |      12 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

**HASH must be identical on all three nodes.** Write down the HASH and REVISION — they are the proof-of-correctness targets for the restore.

---

# Part 5 — The Restore Procedure

## 15. Stage A — Save Current Manifest (Read-Only Safety)

On **each CP**.

**Command:**

```bash
sudo cp /etc/kubernetes/manifests/etcd.yaml ~/etcd.yaml.before-restore
sudo ls -l /etc/kubernetes/pki/etcd/ > ~/etcd-certs.before-restore.txt
```

Read-only. If anything looks different after the restore, we can compare.

## 16. Stage B — Stop kubelet on All Control Planes

**Open three SSH sessions — one per CP.** In each.

**Command:**

```bash
sudo systemctl stop kubelet
```

Wait 30 seconds. Then verify.

**Command:**

```bash
sudo systemctl is-active kubelet
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler' || echo "all gone"
```

**Expected output:**

```
inactive
```

```
all gone
```

### ⚠️ Real-World Gotcha — static pods often do NOT stop promptly

In our lab, 4+ minutes after `systemctl stop kubelet`, the etcd and apiserver containers were **still Running**. This is a known behavior: kubelet's static pod shutdown grace period can be long, and in some kubeadm setups the containers don't receive the stop signal cleanly.

**You must force-stop the containers manually.**

## 17. Force-Stopping Static Pod Containers (The crictl-by-ID Gotcha)

`crictl stop <pod-name>` is **unreliable** — in our lab, passing the pod name returned a container ID but did not actually stop the container. **Use the container ID.**

### Step 1 — List what's running

**Command:**

```bash
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler'
```

**Output example:**

```
00ef7bbb9b6ca   ...   Running   kube-apiserver            7   ...   kube-apiserver-cp01.faekcorp.lab
85dc83466918a   ...   Running   kube-scheduler            9   ...   kube-scheduler-cp01.faekcorp.lab
441fce6d4945b   ...   Running   kube-controller-manager   8   ...   kube-controller-manager-cp01.faekcorp.lab
71f1a8c495cf5   ...   Running   etcd                      7   ...   etcd-cp01.faekcorp.lab
```

The **leftmost column is the container ID**.

### Step 2 — Stop each by ID

**Command:**

```bash
sudo crictl stop 00ef7bbb9b6ca
sudo crictl stop 71f1a8c495cf5
sudo crictl stop 441fce6d4945b
sudo crictl stop 85dc83466918a
```

Or in one shot.

**Command:**

```bash
sudo crictl stop <id1> <id2> <id3> <id4>
```

### Step 3 — Verify all gone

**Command:**

```bash
sudo crictl ps | grep -E 'etcd|apiserver|controller-manager|scheduler' || echo "all gone"
```

**Expected output:**

```
all gone
```

You want `all gone` before proceeding.

### Step 4 — If any still show Running after 10 s, force-remove

**Command:**

```bash
sudo crictl rm -f <container-id>
```

### Step 5 — Confirm no process is bound to the etcd ports

**Command:**

```bash
sudo ss -tlnp | grep -E '2379|2380' || echo "ports 2379/2380 free"
sudo ps aux | grep -E '[e]tcd' || echo "no etcd process"
```

**Expected output:**

```
ports 2379/2380 free
no etcd process
```

**Both must be clean** before proceeding. If a stray etcd process is bound to 2379/2380, find and kill it.

**Command:**

```bash
sudo ps aux | grep -E '[e]tcd'
sudo kill <pid>
```

Use `kill -9` if needed.

> **Why stop all four pods, not just etcd?** Only etcd touches `/var/lib/etcd/` on disk. But stopping all four leaves each node in a clean, identical state — no orphaned apiserver trying to reconnect to a half-restored etcd.

## 18. Stage C — Preserve the Current Data Directories

On **each CP**.

**Command:**

```bash
sudo mv /var/lib/etcd /var/lib/etcd.backup.$(date +%Y%m%d-%H%M%S)
sudo ls -ld /var/lib/etcd*
```

**Expected output:**

```
drwx------ 3 root root 4096 Oct  9 13:15 /var/lib/etcd.backup.20261009-153203
```

Only the `.backup.<timestamp>` directory remains — `/var/lib/etcd` is gone.

**Why `mv` and not `rm`:** this is our rollback safety net. If the restore fails, we put the directory back and we're where we started. Never delete.

## 19. Stage D — Restore on Each Node

Run on **each CP**. The `--initial-cluster` string is **identical on all three**. Only `--name` and `--initial-advertise-peer-urls` differ.

### On cp01

**Command:**

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp01.faekcorp.lab \
  --initial-cluster=cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380 \
  --initial-advertise-peer-urls=https://10.70.21.6:2380 \
  --data-dir=/var/lib/etcd
```

### On cp02

**Command:**

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp02.faekcorp.lab \
  --initial-cluster=cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380 \
  --initial-advertise-peer-urls=https://10.70.31.209:2380 \
  --data-dir=/var/lib/etcd
```

### On cp03

**Command:**

```bash
sudo etcdutl snapshot restore ~/s3-etcd-backups/etcd-snapshot-cp01-20261006-223331.db \
  --name=cp03.faekcorp.lab \
  --initial-cluster=cp01.faekcorp.lab=https://10.70.21.6:2380,cp02.faekcorp.lab=https://10.70.31.209:2380,cp03.faekcorp.lab=https://10.70.41.250:2380 \
  --initial-advertise-peer-urls=https://10.70.41.250:2380 \
  --data-dir=/var/lib/etcd
```

> If you used Path B (laptop hop), replace the source path with `~/etcd-snapshot-cp01-20261006-223331.db`. Everything else is identical.

### Expected output

```
2026-10-09T15:45:53Z  info  snapshot/v3_snapshot.go:305  restoring snapshot  {"path": "...", "wal-dir": "/var/lib/etcd/member/wal", ...}
2026-10-09T15:45:53Z  info  bbolt  Opening db file (/var/lib/etcd/member/snap/db) ...
2026-10-09T15:45:53Z  info  schema/membership.go:138  Trimming membership information from the backend...
2026-10-09T15:45:53Z  info  membership/cluster.go:424  added member  {"cluster-id": "7142046e27c4441c", "added-peer-id": "6108a621ed6af19b", "added-peer-peer-urls": ["https://10.70.21.6:2380"], "added-peer-is-learner": false}
2026-10-09T15:45:53Z  info  membership/cluster.go:424  added member  {"cluster-id": "7142046e27c4441c", "added-peer-id": "7296fe26dd7ee1e2", "added-peer-peer-urls": ["https://10.70.31.209:2380"], "added-peer-is-learner": false}
2026-10-09T15:45:53Z  info  membership/cluster.go:424  added member  {"cluster-id": "7142046e27c4441c", "added-peer-id": "e6ce3932a3a2472a", "added-peer-peer-urls": ["https://10.70.41.250:2380"], "added-peer-is-learner": false}
2026-10-09T15:45:53Z  info  snapshot/v3_snapshot.go:333  restored snapshot  {"path": "...", "data-dir": "/var/lib/etcd", ...}
```

**Key things to verify in the output:**

- `"cluster-id"` is **the same on all three nodes** (`7142046e27c4441c` in our run)
- All three peer URLs are listed
- Final line says `restored snapshot`

If the cluster IDs don't match across nodes, stop and re-run — a mismatch means they will not form one cluster.

### Verify the new directory exists

**Command:**

```bash
sudo ls -ld /var/lib/etcd*
```

**Expected output:**

```
drwx------ 3 root root 4096 Oct  9 15:45 /var/lib/etcd
drwx------ 3 root root 4096 Oct  9 13:15 /var/lib/etcd.backup.20261009-153203
```

Both the new `/var/lib/etcd` and the old `.backup.<timestamp>` present.

## 20. Stage E — Fix Permissions, Restart kubelet

On **each CP**.

**Command:**

```bash
sudo chmod 700 /var/lib/etcd
sudo chown -R root:root /var/lib/etcd
sudo ls -ld /var/lib/etcd
sudo systemctl start kubelet
```

**Expected `ls -ld` output:**

```
drwx------ ... root root ... /var/lib/etcd
```

`etcdutl` usually sets this correctly already, but we run it as belt-and-braces. If permissions are wrong, etcd fails to start with a permission error.

**Note about workers:** Worker nodes (worker1, worker2) are **not touched** by the restore procedure. Their kubelet keeps running throughout. They just can't reach the API server while etcd is down. Once etcd reforms, they reconnect automatically. But see Section 26 — their local container runtime state may need a kubelet restart to reconcile.

## 21. Stage F — Watch Quorum Reform

On cp01 (or bastion).

**Command:**

```bash
kubectl get pods -n kube-system -l component=etcd -w
```

**Output (during reformation):**

```
etcd-cp01.faekcorp.lab   0/1   ContainerCreating   0   5s
etcd-cp02.faekcorp.lab   0/1   ContainerCreating   0   3s
etcd-cp03.faekcorp.lab   0/1   ContainerCreating   0   2s
```

**Output (after ~30–90 seconds):**

```
etcd-cp01.faekcorp.lab   1/1   Running   0   35s
etcd-cp02.faekcorp.lab   1/1   Running   0   33s
etcd-cp03.faekcorp.lab   1/1   Running   0   32s
```

Ctrl+C once all three show `Running 1/1`.

**If it takes longer than 2 minutes**, don't panic — leader election and WAL replay can take a bit. **If after 5 minutes only 1–2 come up**, check logs.

**Command:**

```bash
kubectl logs -n kube-system etcd-cp01.faekcorp.lab --tail=50
```

Common stall causes:
- Peers can't reach each other (network / firewall)
- Cluster IDs mismatch (should not happen — we verified)
- A node got a different `--initial-cluster` (should not happen — we used the same string)

---

# Part 6 — Validating the Restore

## 22. etcd Cluster Health

**Command:**

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

**Expected output — members:**

```
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |       NAME        |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
| 574699e63127f8c0 | started | cp03.faekcorp.lab | https://10.70.41.250:2380 | https://10.70.41.250:2379 |      false |
| 6108a621ed6af19b | started | cp01.faekcorp.lab |   https://10.70.21.6:2380 |   https://10.70.21.6:2379 |      false |
| ee6b7ea6225a4e3e | started | cp02.faekcorp.lab | https://10.70.31.209:2380 | https://10.70.31.209:2379 |      false |
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
```

**Expected output — health:**

```
+------------------------+--------+------------+-------+
|        ENDPOINT        | HEALTH |    TOOK    | ERROR |
+------------------------+--------+------------+-------+
| https://127.0.0.1:2379 |   true | 8.515342ms |       |
+------------------------+--------+------------+-------+
```

> **Note:** member IDs will be **different** from before the restore (e.g. `7296fe26dd7ee1e2` replaced `ee6b7ea6225a4e3e`). This is normal — restore creates fresh member IDs because it builds a new cluster identity. The names (cp01.faekcorp.lab etc.) remain the same.

## 23. Proving Point-in-Time Correctness

### 23.1 — The canary test (strongest proof)

Before the restore, we created two namespaces **after** the snapshot moment.

**Command (before restore):**

```bash
kubectl create namespace sim-backup
kubectl create namespace after-snapshot
kubectl create deploy sim-deploy --image=nginx --replicas 2 -n sim-backup
kubectl create deploy after-snapshot-deploy --image=nginx --replicas 2 -n after-snapshot
```

After restore, these must be **gone**.

**Command (after restore):**

```bash
kubectl get ns sim-backup
kubectl get ns after-snapshot
kubectl get ns
```

**Output:**

```
Error from server (NotFound): namespaces "sim-backup" not found
Error from server (NotFound): namespaces "after-snapshot" not found
```

```
NAME              STATUS   AGE
cilium-secrets    Active   5d17h
default           Active   5d23h
kube-node-lease   Active   5d23h
kube-public       Active   5d23h
kube-system       Active   5d23h
```

Both canaries gone, five originals intact. Point-in-time restore confirmed.

### 23.2 — The revision test (numerical proof)

**Command:**

```bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w table
```

Look at the `REVISION` column. It should be **just above the snapshot's revision** (`214775` + a few hundred for writes since restore). If it's back to ~214775, point-in-time rollback is confirmed.

### 23.3 — Original workloads check

**Command:**

```bash
kubectl get deploy -n default
kubectl get deploy -n kube-system
kubectl get ds -n kube-system
kubectl get ns cilium-secrets
```

**Expected output:**

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   2/2     2            2           3d19h
```

```
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
coredns           2/2     2            2           5d23h
cilium-operator   1/1     1            1           5d17h
```

```
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   AGE
cilium         5         5         5       5            5           5d17h
cilium-envoy   5         5         5       5            5           5d17h
```

All originals intact with their original AGE values preserved — proof they were restored from the snapshot, not freshly created.

## 24. Workload and Scheduling Tests

**Command:**

```bash
kubectl run test-restore --image=nginx:latest --restart=Never
kubectl get pod test-restore
kubectl delete pod test-restore
```

**Expected output:**

```
pod/test-restore created
```

```
NAME           READY   STATUS    RESTARTS   AGE
test-restore   1/1     Running   0          10s
```

If the pod reaches `Running 1/1`, then:
- Scheduler works
- Kubelet can pull and run images
- CNI (Cilium) is functional
- apiserver accepts writes

## 25. The cilium-operator CrashLoop That Self-Heals

### What we observed

A few minutes after the restore completed.

**Command:**

```bash
kubectl get all -A
```

**Output (excerpt):**

```
kube-system   pod/cilium-operator-56dcd959f4-4vdpc   0/1   CrashLoopBackOff   25 (2m39s ago)   3d22h
kube-system   deployment.apps/cilium-operator       0/1   1   0   5d17h
```

Deployment: `0/1 AVAILABLE`, pod restarts climbing.

### What we did

**Nothing.** No pod deletion, no deployment restart.

### What happened

Within a few minutes, the same pod stabilized on its own.

**Command:**

```bash
kubectl get all -A
```

**Output (excerpt):**

```
kube-system   pod/cilium-operator-56dcd959f4-4vdpc   1/1   Running   26 (8m ago)   3d22h
kube-system   deployment.apps/cilium-operator       1/1   1   1   5d17h
```

`0/1` → `1/1`. `AVAILABLE 1`. The CrashLoop ended.

### Why it self-heals

- `cilium-operator` uses a leader-election **Lease** stored in etcd
- After restore, the Lease from before the snapshot is "stale"
- The operator's restart loop **eventually succeeds** at renewing or re-acquiring the Lease
- Once it does, the crash loop stops on its own

There is no coordination protocol failure — it's a transient state where the operator and the stored Lease disagree briefly. Both converge within a few minutes.

### What to do in production

| Observation | Action |
|-------------|--------|
| CrashLoop lasts **< 5 minutes** | Wait. Nothing to do. |
| CrashLoop persists **> 10 minutes** | Check logs: `kubectl logs -n kube-system deploy/cilium-operator --tail=50` |
| Log shows a **permanent** error | Investigate that specific error. May be unrelated to the restore. |
| Only if stuck after 15+ minutes | Consider `kubectl rollout restart deployment -n kube-system cilium-operator` |

**Default posture after a restore is: wait and observe.** Do not reflexively delete pods. The restore changes the entire cluster state, and many controllers will need a few minutes to re-sync.

Any controller that uses Kubernetes `coordination.k8s.io/Lease` objects — Cilium, cert-manager, cloud controllers, scheduler leader-election — may exhibit a temporary CrashLoop after a restore. **Most self-heal.**

## 26. Worker Nodes After the Restore — Stale Containers and Pending Pods

Worker nodes have **no etcd** and are not touched by the restore procedure itself. Their kubelet keeps running the entire time. They only lose API connectivity during the 3–5 minute window when etcd is down.

**But this does not mean workers are unaffected after the restore.** The restore changes the *control-plane* state — what the API knows — but the worker nodes still hold their *local runtime* state (containers, sandboxes) in containerd. These two can temporarily diverge.

### What we actually observed

Both workers existed in the cluster **before** the snapshot. Nothing was added or rejoined after the restore. All five nodes were `Ready` shortly after the restore completed.

However, `crictl ps` on the workers still showed containers running for Pods that no longer existed in the restored API state.

**On worker1:**

```
NAME       POD                                      NAMESPACE
nginx      after-snapshot-deploy-798d66d75b-xscvq   after-snapshot
nginx      sim-deploy-7d949546-f75cj                sim-backup
nginx      nginx-deploy-8b9dbd8c9-tjwgb             default
```

**On worker2:**

```
NAME       POD                                      NAMESPACE
nginx      after-snapshot-deploy-798d66d75b-rvr4g   after-snapshot
nginx      sim-deploy-7d949546-9kcs5                sim-backup
nginx      nginx-deploy-8b9dbd8c9-brsgn             default
```

Meanwhile, `kubectl get ns` on the bastion no longer listed `after-snapshot` or `sim-backup` — the etcd restore had wiped them from the API.

**Contradiction:** API says the namespaces are gone. containerd says their Pods' containers are still running.

### Why this happens

etcd and containerd maintain **different types of state**:

| Component | Responsibility |
|-----------|---------------|
| etcd | Stores Kubernetes cluster state (API objects) |
| kube-apiserver | Exposes the API backed by the restored state |
| kubelet | Reconciles Pods assigned to a node against the desired state |
| containerd | Manages local container and Pod sandbox state |

Restoring etcd changes the **cluster state**. It does **not** execute a cleanup operation against containerd on every worker.

Under normal operation, kubelet continuously reconciles its local runtime against the API. After a large etcd rollback, kubelet should eventually notice that these Pods are gone and clean up their containers. In our experiment, some stale containers kept running longer than expected.

**Key finding:** restoring etcd and cleaning up worker runtime state are **separate operations**.

### What we did about it

After confirming the stale containers were present, we restarted kubelet on the affected worker.

**Command (on worker2):**

```bash
sudo systemctl restart kubelet.service
```

**Immediately after**, `crictl ps` on worker2 no longer listed the `after-snapshot` NGINX container. The `sim-backup` container was still there at that instant. On a later check, that container and CoreDNS had also disappeared from the running list. Cilium agent, Cilium Envoy, Cilium operator, and the default-namespace NGINX container were still running.

The change was consistent with kubelet reconciling local runtime state after its restart.

### A second symptom — new Pods stuck in Pending

While investigating, we created a new Deployment to verify the cluster could accept new workloads.

**Command (on bastion):**

```bash
kubectl create ns after-restore
kubectl create deploy after-restore-deploy --image=nginx --replicas=2 -n after-restore
```

The Deployment was created, but its Pods remained `Pending`:

```
NAME                                    READY   STATUS
after-restore-deploy-759c7fb75b-fvckk   0/1     Pending
after-restore-deploy-759c7fb75b-l62j6   0/1     Pending
```

`kubectl describe pod` showed the scheduler had successfully assigned the Pods to worker1 and worker2 (`PodScheduled: True`) — so scheduling was not the problem. The failure was **later in the startup path**, on the workers.

After restarting kubelet on worker2, we created a **new** Deployment, and this time both Pods reached `Running`:

```
NAME                                    READY   STATUS    RESTARTS
after-restore-deploy-759c7fb75b-qr6zg   1/1     Running   0
after-restore-deploy-759c7fb75b-xf4xl   1/1     Running   0
```

`deployment.apps/after-restore-deploy   2/2   2   2`

### What we established vs. what we did not

**Established:**

1. The restored cluster accepted a new namespace and Deployment
2. The scheduler could assign Pods to worker nodes
3. Restarting kubelet on worker2 was followed by the disappearance of some stale containers from the running-container list
4. A subsequent two-replica NGINX Deployment reached `2/2` available replicas

**Not conclusively established:**

- That the kubelet restart alone *caused* the Pending Pods to resolve (the successful result came from a *newly created* Deployment)
- The exact reason the original Pending Pods never became Ready — we did not capture enough runtime logs (no `crictl ps -a`, no `crictl pods -a`, no kubelet journal at the right moment)

So the correct framing is: **restarting kubelet on the affected worker and retesting with a new Deployment resolved the observed condition** — not "restart kubelet on every worker after every restore".

### Recommended verification for workers after a restore

Do not assume the restore cleaned up worker runtime state. Verify explicitly.

**Phase A — Restored API state (on bastion):**

```bash
kubectl get nodes
kubectl get ns
kubectl get deployments -A
kubectl get pods -A -o wide
```

**Phase B — Worker runtime state (on each worker):**

```bash
sudo systemctl status kubelet --no-pager
sudo systemctl status containerd --no-pager
sudo crictl ps -a
sudo crictl pods -a
```

Use `-a` — checking only *running* containers is not enough to distinguish running, exited, and absent runtime objects.

**Phase C — If stale containers persist or new Pods remain Pending:**

```bash
sudo journalctl -u kubelet --since "30 minutes ago" --no-pager
```

If kubelet is not reconciling correctly, restart it on the affected worker only:

```bash
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

Recheck runtime state and logs. **Do not restart every worker simultaneously — investigate and recover one worker at a time.**

**Phase D — Retest with a fresh workload:**

```bash
kubectl create namespace restore-validation
kubectl create deployment nginx-test --image=nginx --replicas=2 -n restore-validation
kubectl rollout status deployment/nginx-test -n restore-validation
kubectl get pods -n restore-validation -o wide
```

Expected: two Pods `Running`, `2/2` available. Clean up:

```bash
kubectl delete namespace restore-validation
```

### Worker section — summary

| Item | Finding |
|------|---------|
| Workers added after snapshot? | ❌ No — both existed before |
| Worker Node objects restored? | ✅ Yes, all `Ready` post-restore |
| Stale containers observed on workers? | ✅ Yes — Pods from wiped namespaces still in containerd |
| Why? | etcd restore ≠ containerd cleanup; kubelet reconciles separately |
| What fixed it? | Restarting kubelet on the affected worker |
| New workloads ran after? | ✅ Yes, once kubelet reconciliation had run |
| Root cause fully proven? | ❌ No — plausible explanation, not fully captured in logs |

**The main lesson:** an etcd restore recovers **control-plane state**, not a complete picture of every worker's container runtime. After a restore, verify that API state, kubelet, and containerd have converged on the intended workload state. If they haven't, restart kubelet on the affected worker and retest — don't assume the restore handled it.

---

# Part 7 — Post-Restore Cleanup

## 27. What to Keep, What to Delete, When

| Item | Keep for | Why |
|------|----------|-----|
| `~/s3-etcd-backups/<snapshot>.db` on each CP | Until next successful drill | Source of truth for this restore |
| `~/etcd-snapshot-*.db` on each CP (Path B) | Until next successful drill | Same — if using laptop-hop path |
| S3 object `s3://.../etcd-snapshots/<snapshot>.db` | Per retention policy | The durable backup |
| `/var/lib/etcd.backup.<timestamp>` on each CP | **≥24 hours** | Rollback safety if issues surface |
| `~/etcd.yaml.before-restore` | ≥24 hours | Compare manifest if anything odd |
| `~/pre-restore-cluster-state.txt` | ≥24 hours | Diff resource state if needed |

**Delete immediately after use.**

**Command:**

```bash
sudo rm /tmp/snapshot.db
```

Only if you staged one there.

**Delete after 24–48 hours of stable operation.**

**Command:**

```bash
sudo rm -rf /var/lib/etcd.backup.*
```

Run on each CP. **Never `rm` these without verifying the restore first.**

No Kubernetes-level pod deletions or rollout restarts were required for the control-plane components in our drill. On the workers, we restarted kubelet once to force reconciliation — see Section 26.

---

# Part 8 — Production Considerations

## 28. RTO and RPO

For this 3-CP stacked etcd cluster (2 workers):

| Phase | Duration |
|-------|----------|
| Pre-flight + canary creation | 5–10 min |
| Stage B (stop kubelet + force-stop containers) | 2–4 min |
| Stage C (mv data dirs) | < 1 min |
| Stage D (etcdutl restore on 3 nodes) | 1–2 min |
| Stage E (perms + start kubelet) | < 1 min |
| Stage F (quorum reform) | 1–3 min |
| Stage G (validation) | 5–10 min |
| Worker kubelet reconciliation (if needed) | 1–5 min per affected worker |
| Self-heal of Lease-based controllers | 3–15 min (background) |
| **Total RTO (API usable)** | **~15–25 min** |
| **Total RTO (fully settled, workers reconciled)** | **~30–45 min** |

RPO = snapshot frequency. Hourly snapshots → up to 1 hour of data loss. Daily → up to 24 hours.

## 29. Automating Restores (Don't)

**Do not automate restore.** Restore is a high-stakes operation with a human in the loop by design:

- Requires deciding *which* snapshot to restore to
- Requires verifying the cluster's current state before committing
- Requires post-restore validation and reconciliation
- A botched restore can make things worse than the original problem

**What you CAN automate:**
- Backup (systemd timer on cp01)
- Snapshot integrity checks (post-upload verification, alerted)
- Periodic restore drills on an isolated test cluster
- Pre-flight checks (verifying snapshot availability and integrity)

**What stays manual:**
- The destructive `mv /var/lib/etcd` step
- The `etcdutl snapshot restore` command on each node
- The decision to proceed at each stage
- The decision to restart kubelet on a worker if reconciliation stalls

## 30. Regular Restore Drills

The single most valuable practice: **actually restore, on a schedule, on a non-production cluster**.

A backup that has never been restored is only an assumption. Run this full procedure monthly (or quarterly) against a staging cluster with the same topology. Time it. Document the RTO. Fix surprises before they hit production.

**Specifically for workers:** during a drill, capture `crictl ps -a`, `crictl pods -a`, and `journalctl -u kubelet` on each worker **before, during, and after** the restore. This is the strongest evidence for how kubelet reconciles after a rollback, and it turns "restart kubelet and retest" into a properly understood procedure.

---

# Part 9 — Reference

## 31. Complete Command Reference

### Snapshot verification

| Command | Purpose |
|---------|---------|
| `etcdutl snapshot status <path> -w table` | Verify snapshot integrity (offline) |
| `sha256sum <path>` | Cross-check with hash |
| `ls -lh <path>` | Confirm size |

### etcd inspection (while running)

| Command | Purpose |
|---------|---------|
| `etcdctl member list -w table` | Cluster membership |
| `etcdctl endpoint health -w table` | Per-endpoint health |
| `etcdctl endpoint status -w table` | Revision, size, leader info |

### Restore workflow

| Stage | Command |
|-------|---------|
| A | `sudo cp /etc/kubernetes/manifests/etcd.yaml ~/etcd.yaml.before-restore` |
| B | `sudo systemctl stop kubelet` |
| B | `sudo crictl ps \| grep -E 'etcd\|apiserver\|...'` |
| B | `sudo crictl stop <container-id>` |
| B | `sudo crictl rm -f <container-id>` (if needed) |
| B | `sudo ss -tlnp \| grep -E '2379\|2380'` (must be free) |
| C | `sudo mv /var/lib/etcd /var/lib/etcd.backup.$(date +%Y%m%d-%H%M%S)` |
| D | `sudo etcdutl snapshot restore <path> --name=... --initial-cluster=... --initial-advertise-peer-urls=... --data-dir=/var/lib/etcd` |
| E | `sudo chmod 700 /var/lib/etcd && sudo chown -R root:root /var/lib/etcd` |
| E | `sudo systemctl start kubelet` |
| F | `kubectl get pods -n kube-system -l component=etcd -w` |
| G | (see validation section) |
| Worker | `sudo systemctl restart kubelet` (only if reconciliation stalls) |

### AWS CLI

| Command | Purpose |
|---------|---------|
| `aws sts get-caller-identity` | Verify IAM role |
| `aws s3 ls s3://<bucket>/etcd-snapshots/` | List available snapshots |
| `aws s3 cp s3://<bucket>/etcd-snapshots/<file> <dir>/` | Download |

### SSH / SCP (bastion hop)

| Command | Purpose |
|---------|---------|
| `scp -i <cp-key> -o ProxyCommand="ssh -i <bastion-key> -W %h:%p ubuntu@<bastion>" <local> <dest>` | Copy via bastion tunnel |
| `-o IdentitiesOnly=yes` | Force use of specified key only |

### Worker runtime inspection

| Command | Purpose |
|---------|---------|
| `sudo crictl ps -a` | All containers (running + exited) |
| `sudo crictl pods -a` | All pod sandboxes |
| `sudo systemctl status kubelet --no-pager` | kubelet state |
| `sudo systemctl status containerd --no-pager` | containerd state |
| `sudo journalctl -u kubelet --since "30 minutes ago" --no-pager` | kubelet logs |

## 32. Key Paths

| Path | What it is |
|------|-----------|
| `/etc/kubernetes/manifests/etcd.yaml` | etcd static pod manifest |
| `/etc/kubernetes/pki/etcd/ca.crt` | etcd cluster CA cert |
| `/etc/kubernetes/pki/etcd/server.crt` | etcd server cert |
| `/etc/kubernetes/pki/etcd/server.key` | etcd server private key |
| `/var/lib/etcd/` | Live etcd data directory |
| `/var/lib/etcd/member/snap/db` | Live bbolt database |
| `/var/lib/etcd/member/wal/` | Write-ahead log |
| `/var/lib/etcd.backup.<ts>/` | Pre-restore data (rollback safety) |
| `~/s3-etcd-backups/` | Downloaded snapshots, ready to restore (Path A) |
| `~/etcd-snapshot-*.db` | Snapshots copied via laptop hop (Path B) |
| `/usr/local/bin/etcdctl` | etcd CLI client |
| `/usr/local/bin/etcdutl` | etcd offline utility |
| `~/.aws/credentials` | Should NOT exist on cp01/02/03 (use IAM role) |

## 33. Troubleshooting

### etcd pods fail to start after restore

**Command:**

```bash
kubectl logs -n kube-system etcd-cp01.faekcorp.lab --tail=50
```

Common causes:
- **Permissions** — fix with `chmod 700` + `chown -R root:root`
- **Mismatched `--initial-cluster`** — verify all three restore commands used the identical string
- **Cert issues** — verify `/etc/kubernetes/pki/etcd/` files exist
- **Port already in use** — a stray etcd process; check `sudo ss -tlnp | grep 2379`

### Cluster loses quorum after restore

- Verify all nodes restored from the **same snapshot** (same HASH)
- Verify **all three `--initial-cluster` strings are identical**
- Check each node's `cluster-id` in the restore log — must match across nodes

### API server reports connection refused

Wait 3–5 minutes. The apiserver needs time to discover the reformed etcd, establish connections, and initialize. If it persists.

**Command:**

```bash
kubectl logs -n kube-system kube-apiserver-cp01.faekcorp.lab --tail=50
```

### Resources missing after restoration

- **Using wrong snapshot** — check REVISION and TOTAL KEYS
- **Old snapshot** — expected; you lost data between snapshot time and restore time
- **Pre-snapshot missing** — the restore itself failed; check etcd logs

### Cannot connect to etcd with etcdctl

- Verify certs exist: `sudo ls -l /etc/kubernetes/pki/etcd/{ca.crt,server.crt,server.key}`
- Use the correct endpoint: `--endpoints=https://127.0.0.1:2379`
- Need `sudo` for cert read access

### cilium-operator (or similar) CrashLoopBackOff after restore

Expected behavior; usually **self-heals within a few minutes**. Observe, don't intervene prematurely.

**Command:**

```bash
kubectl get pods -n kube-system -w
```

Only escalate (`kubectl rollout restart`) if stuck > 15 minutes.

### Worker has stale containers from deleted namespaces

Expected after a large etcd rollback. containerd doesn't clean itself up just because the API state changed.

**Command (on the workers):**

```bash
sudo crictl ps -a
sudo crictl pods -a
sudo journalctl -u kubelet --since "30 minutes ago" --no-pager
```

If kubelet isn't reconciling, restart the workers:

```bash
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

Recheck `crictl ps` and retest with a fresh Deployment.

### Worker has new Pods stuck in Pending

Scheduler assigned them (check `kubectl describe pod` → `PodScheduled: True`), but they don't start. This points at kubelet reconciliation or containerd sandbox creation on the worker.

**Command (on the workers):**

```bash
sudo journalctl -u kubelet --since "10 minutes ago" --no-pager
sudo systemctl restart kubelet
```

Then retest with a **new** Deployment — do not assume the original Pending Pods will resolve.

**Command (on the workers):**

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

Common causes:
- Worker's kubelet cert expired during the window
- Worker can't reach the API (check HAProxy / network)
- Worker's kubelet is running but not reconciling — restart it

### `crictl stop <pod-name>` doesn't stop the container

Use the **container ID** (leftmost column of `crictl ps`), not the pod name.

**Command:**

```bash
sudo crictl ps | grep etcd
sudo crictl stop <container-id>
sudo crictl rm -f <container-id>
```

Use `rm -f` if it's still running.

### Containers still running after `systemctl stop kubelet`

Same as above — kubelet's shutdown grace period can be long; force-stop with crictl by ID.

### Direct `scp` from cp01 to cp02/cp03 fails with "Permission denied (publickey)"

The CPs cannot SSH to each other in this topology. Use the laptop-hop pattern with `ProxyCommand` through the bastion.

**Command (on the laptop):**

```bash
scp -o IdentitiesOnly=yes \
    -i ~/.ssh/lab-controlplane-keypair.pem \
    -o ProxyCommand="ssh -i ~/.ssh/lab-bastion-keypair.pem -o IdentitiesOnly=yes -W %h:%p ubuntu@100.61.170.135" \
    <local-file> \
    ubuntu@<dest-cp-ip>:~/
```

**Output (first run only):**

```
The authenticity of host '<dest-cp-ip> (<no hostip for proxy command>)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '<dest-cp-ip> (<no hostip for proxy command>)' to the list of known hosts.
<file>    100%   <size>  <rate>  <time>
```

## 34. Summary — The Whole Flow at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│  Pre-flight  │  Verify snapshot on all 3 CPs, same HASH       │
├──────────────────────────────────────────────────────────────┤
│  Canaries    │  Create post-snapshot namespaces + deployments │
├──────────────────────────────────────────────────────────────┤
│  Stage 0     │  Snapshot source (either path works):          │
│              │   A) aws s3 cp → ~/s3-etcd-backups/            │
│              │   B) scp via laptop → bastion → cp02/cp03      │
│              │  Verify HASH identical on all 3 CPs            │
├──────────────────────────────────────────────────────────────┤
│  Stage A     │  Save current manifest (read-only safety)      │
├──────────────────────────────────────────────────────────────┤
│  Stage B     │  Stop kubelet (all 3 CPs)                      │
│              │  Force-stop containers via crictl (by ID)      │
│              │  Verify ports 2379/2380 free                   │
├──────────────────────────────────────────────────────────────┤
│  Stage C     │  mv /var/lib/etcd → /var/lib/etcd.backup.<ts>  │
├──────────────────────────────────────────────────────────────┤
│  Stage D     │  etcdutl snapshot restore on each node         │
│              │  Same --initial-cluster, node-specific --name  │
│              │  Verify cluster-id identical across all 3      │
├──────────────────────────────────────────────────────────────┤
│  Stage E     │  chmod 700, chown root, start kubelet          │
├──────────────────────────────────────────────────────────────┤
│  Stage F     │  Watch etcd pods reform quorum                 │
├──────────────────────────────────────────────────────────────┤
│  Stage G     │  Validate: etcd healthy, canaries gone,        │
│              │  originals intact, workers Ready, cluster      │
│              │  schedulable                                   │
├──────────────────────────────────────────────────────────────┤
│  Stage H     │  Wait 5–15 min — Lease-based controllers       │
│              │  (cilium-operator etc.) may CrashLoop briefly  │
│              │  and then self-heal. Do NOT delete pods        │
│              │  unless stuck > 15 min.                        │
│              │                                                │
│              │  Check workers: stale containers may remain    │
│              │  from wiped namespaces. If new Pods stay       │
│              │  Pending, restart kubelet on the affected      │
│              │  worker and retest with a fresh Deployment.    │
│              │                                                │
│              │  Keep .backup dirs ≥24h, then clean up         │
└──────────────────────────────────────────────────────────────┘
```

### The key mental model

```
              BACKUP                                 RESTORE
              ──────                                 ───────

    live cluster                         portable snapshot file
         │                                       │
         ▼                                       ▼
   etcdctl snapshot save                 etcdutl snapshot restore
         │                                       │
         ▼                                       ▼
   portable dump file                   fresh etcd data dir
   (single .db, dated)                  (member/snap/db + wal + new cluster-id)
         │                                       │
         ▼                                       ▼
    uploaded to S3                     etcd starts, joins new cluster
         │                                       │
         ▼                                       ▼
  downloaded to each CP              quorum reforms, API returns,
  (direct S3 or laptop hop)          workers reconnect — verify their
                                     local runtime state separately
```

**Backup = "export the cluster's brain into a file."**
**Restore = "build a fresh cluster body on this node, and pour that brain into it."**

**Post-restore = "check that every worker's body agrees with the restored brain."**

---
