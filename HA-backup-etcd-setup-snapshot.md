# Kubernetes etcd Backup Setup & S3 Upload — Complete Runbook

A hands-on, step-by-step guide for backing up an etcd cluster on a kubeadm-based Kubernetes cluster, verifying the snapshot, and uploading it to S3 using an IAM role.

Written from real experiments on a 3-control-plane HA cluster:
- **Kubernetes:** v1.35.8
- **etcd:** 3.6.6 (image `registry.k8s.io/etcd:3.6.6-0`)
- **Cluster name:** faekcorp-lab
- **Nodes:** cp01, cp02, cp03 (stacked etcd) + worker nodes
- **Load balancer:** HAProxy at `haproxy.faekcorp.lab:6443`
- **Bastion:** kubectl-only admin host
- **S3 bucket:** `ha-cluster-s3-etcd-lab` (us-east-1)

---

## Table of Contents

**Part 1 — Why etcd Backup Matters**
1. The mental model
2. HA is not backup
3. What the backup contains (and what it doesn't)

**Part 2 — The Architecture We're Working With**
4. Cluster topology
5. Where the tools and data live

**Part 3 — Understanding `/var/lib/etcd/`**
6. The three-layer model
7. WAL, snapshots, and the live database
8. The three different "snapshots" you'll encounter

**Part 4 — Where to Install the Tools**
9. Bastion vs control-plane: the decision
10. Installing etcdctl and etcdutl on cp01

**Part 5 — Creating and Verifying the Snapshot**
11. Step 1 — Validate the etcd cluster
12. Step 2 — Take the snapshot
13. Step 3 — Verify file presence
14. Step 4 — Verify snapshot integrity
15. Step 5 — Copy off the node
16. Snapshot naming and retention

**Part 6 — Uploading to S3 with IAM Role**
17. Why IAM roles, not static keys
18. The two identities: you vs the instance
19. What goes where
20. Console setup steps
21. CLI alternative
22. Installing AWS CLI on cp01 (no configuration needed)
23. Verifying the role works
24. The upload

**Part 7 — Production Considerations**
25. Automation and scheduling
26. Least-privilege IAM policy
27. Retention and lifecycle
28. Monitoring and alerting
29. What's not covered

**Part 8 — Reference**
30. Complete command reference
31. Key paths
32. Summary — the whole flow at a glance

---

# Part 1 — Why etcd Backup Matters

## 1. The Mental Model

etcd is the **single source of truth** for your entire Kubernetes cluster. Every object — Namespaces, Deployments, Services, Secrets, ConfigMaps, RBAC rules, Nodes, Leases, CRDs — lives in etcd as a key-value pair.

```
                  kubectl get pods
                         │
                         ▼
                  kube-apiserver
                         │
                         ▼
                       etcd
                         │
                         ▼
               /var/lib/etcd/  ← the physical disk state
```

If etcd is lost or corrupted:
- The cluster has no memory of what it was supposed to run
- Worker nodes only run what they're told — they cannot reconstruct state
- Every YAML applied over months is gone

## 2. HA Is Not Backup

This is the single most important distinction.

| Concern | Solution |
|---------|----------|
| Node failure | HA (3-member etcd with Raft) |
| Bad data | Backup (external snapshot) |
| Accidental deletion | Backup |
| Ransomware / corruption | Backup |
| Operator mistake (`kubectl delete namespace production`) | Backup |

**Raft replicates bad data as faithfully as good data.** If you delete a namespace, all three etcd members happily replicate the deletion. HA protects against *hardware failure*, not *data loss*.

```
                    Raft replication
       cp01  ◄──────────────────►  cp02
         ▲                            ▲
         │                            │
         └──────────┬─────────────────┘
                    │
                    ▼
                  cp03

       Delete a namespace → all three members delete it
       HA works perfectly — the data is still gone
```

**Backups live outside the cluster. That's the only way to recover from data loss.**

## 3. What the Backup Contains (and What It Doesn't)

**Contains:**

- Every Namespace, Pod, Deployment, Service, ConfigMap, Secret
- ServiceAccounts, Roles, RoleBindings, ClusterRoles, ClusterRoleBindings
- CRDs and their instances
- Nodes, Leases, Events (recent)
- Anything else stored as a Kubernetes object

**Does NOT contain:**

- Container images
- Application data on PersistentVolumes (databases, file uploads)
- Cloud provider resources (ELBs, EBS volumes, S3 buckets)
- External systems (DNS, certificate managers)
- etcd PKI certificates themselves

You need **separate backup strategies** for each of those.

---

# Part 2 — The Architecture We're Working With

## 4. Cluster Topology

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
```

Key facts:

- **3 control-plane nodes** with **stacked etcd** (etcd runs on the same nodes)
- etcd runs as a **static pod** (managed by kubelet, not a Deployment)
- etcd uses **mTLS** — client certs are required
- Certs live under `/etc/kubernetes/pki/etcd/`
- Data lives under `/var/lib/etcd/`

## 5. Where the Tools and Data Live

| Component | Location | Purpose |
|-----------|----------|---------|
| `kubectl` | Bastion | Kubernetes API client |
| `etcdctl` | cp01 | etcd client (online operations) |
| `etcdutl` | cp01 | etcd offline utility |
| `aws` CLI | cp01 | S3 upload |
| etcd certs | `/etc/kubernetes/pki/etcd/` on each CP | mTLS |
| etcd live data | `/var/lib/etcd/member/` on each CP | etcd internal storage |
| etcd snapshot | `/var/lib/etcd/snapshot.db` (or wherever we choose) | Backup artifact |
| IAM role | Attached to cp01 (via EC2 instance profile) | AWS credentials |

**Design principle:** keep admin tooling as close to the resource as possible, minimize credential distribution.

---

# Part 3 — Understanding `/var/lib/etcd/`

Before touching anything, understand what you're looking at. This directory is one etcd member's local persistent storage.

## 6. The Three-Layer Model

A static pod (like etcd) exists in three independent layers:

```
┌────────────────────────────────────────────────────────┐
│  Layer 1 — Manifest file (on node disk)                │
│                                                        │
│  /etc/kubernetes/manifests/etcd.yaml                   │
│                                                        │
│  Source of truth. Only kubelet reads it.               │
└────────────────────────────────────────────────────────┘
                        ↓  read by kubelet
┌────────────────────────────────────────────────────────┐
│  Layer 2 — Container (in containerd)                   │
│                                                        │
│  The actual running etcd process.                      │
│  Writes to /var/lib/etcd/ and renews leases.           │
└────────────────────────────────────────────────────────┘
                        ↓  registered by kubelet
┌────────────────────────────────────────────────────────┐
│  Layer 3 — Mirror pod (in API server / etcd)           │
│                                                        │
│  A read-only reflection object.                        │
│  kubectl shows you THIS. It is NOT the pod.            │
└────────────────────────────────────────────────────────┘
```

## 7. WAL, Snapshots, and the Live Database

```
/var/lib/etcd/
└── member/
    ├── snap/
    │   ├── db                                ← live key-value database (bbolt)
    │   ├── 0000000000000006-...35b76.snap    ← internal checkpoint
    │   ├── 0000000000000006-...38287.snap
    │   └── ...
    └── wal/
        ├── 0000000000000000-...0000.wal
        ├── 0000000000000001-...2b0e.wal
        ├── 0000000000000002-...ab69.wal      ← currently being written
        └── *.tmp                             ← pre-allocated spares
```

| Path | What it is | Managed by |
|------|-----------|------------|
| `member/` | This member's persistent storage | etcd |
| `snap/` | Snapshot/database storage area | etcd |
| `snap/db` | **Live key-value database** (bbolt format) | etcd |
| `snap/*.snap` | Internal checkpoints (metadata only, ~9 KB each) | etcd |
| `wal/` | Write-ahead log (Raft log entries) | etcd |
| `wal/*.wal` | Individual WAL segments (fixed max size ~64 MB) | etcd |

### The `.snap` filenames

```
0000000000000006-0000000000035b76.snap
└──────┬───────┘ └──────┬───────┘
   Raft term      Raft index
   (hex)          (hex)
```

- **Term** — increments when a new Raft leader is elected
- **Index** — position in the Raft log (each committed operation increments it)

Both are **hexadecimal**, left-padded with zeros.

### Why WAL and internal snapshots both exist

Think of it as a video game with checkpoints:

- **Internal snapshot** = checkpoint (state of the world at index N)
- **WAL** = all moves since the last checkpoint

On startup, etcd:
1. Loads the newest internal snapshot
2. Replays WAL entries newer than that snapshot
3. Result = current state

Without internal snapshots, etcd would have to replay the entire WAL from the beginning of time. With them, only recent WAL entries need replaying.

## 8. The Three Different "Snapshots" You'll Encounter

| Snapshot type | Location | Purpose | Portable? |
|---------------|----------|---------|-----------|
| Internal `.snap` | `member/snap/*.snap` | etcd's own checkpoints | ❌ |
| Live DB | `member/snap/db` | Current on-disk state | ❌ |
| **Backup** (from `etcdctl snapshot save`) | Anywhere you choose | **Disaster recovery artifact** | ✅ |

**Only the third one matters for disaster recovery.**

### Why you can't just `cp db`

1. The file is being actively written by a running etcd
2. bbolt uses memory-mapped I/O — copying a live mmap file risks inconsistency
3. `db` alone doesn't include uncheckpointed WAL entries
4. The `db` format is internal — no supported restore path

`etcdctl snapshot save` asks the running etcd to produce a **consistent, portable** snapshot. That's the supported method.

---

# Part 4 — Where to Install the Tools

## 9. Bastion vs Control-Plane: The Decision

| Location | Install etcdctl? | Why |
|----------|-----------------|-----|
| **Bastion** | ❌ No | Would require copying private keys and opening port 2379 |
| **All three CPs** | ❌ No | Unnecessary duplication |
| **One CP (cp01)** | ✅ **Yes** | Same host as the certs and local etcd |

### The reasoning

`kubectl` and `etcdctl` are **not the same tool**:

| Tool | Talks to | Port |
|------|----------|------|
| `kubectl` | kube-apiserver | 6443 |
| `etcdctl` | etcd directly | 2379 |

Installing `etcdctl` on the bastion would require:

- Copying `/etc/kubernetes/pki/etcd/{ca.crt,server.crt,server.key}` to the bastion — **spreads sensitive private keys across more machines**
- Opening port 2379 from the bastion to the CPs — **widens the attack surface**

Neither is necessary. On cp01, everything is already in the right place.

**Golden rule:** keep admin tooling as close to the resource as possible, and minimize credential distribution.

## 10. Installing etcdctl and etcdutl on cp01

### 10.1 — Determine the running etcd version

**Never assume — always check.** The client must match the server version.

From the bastion:

```bash
kubectl get pods -n kube-system -l component=etcd \
  -o jsonpath='{.items[*].spec.containers[*].image}'; echo
```

Example output:

```
registry.k8s.io/etcd:3.6.6-0 registry.k8s.io/etcd:3.6.6-0 registry.k8s.io/etcd:3.6.6-0
```

The `3.6.6` tells us which etcdctl version to install.

### 10.2 — Download and extract on cp01

```bash
ssh ubuntu@cp01

cd /tmp
wget https://github.com/etcd-io/etcd/releases/download/v3.6.6/etcd-v3.6.6-linux-amd64.tar.gz
tar -xvf etcd-v3.6.6-linux-amd64.tar.gz
```

You get:

```
etcd-v3.6.6-linux-amd64/
├── etcd       ← server binary (not needed)
├── etcdctl    ← CLI client
└── etcdutl    ← offline utility
```

### 10.3 — Install the two tools

```bash
sudo install -m 0755 etcd-v3.6.6-linux-amd64/etcdctl /usr/local/bin/etcdctl
sudo install -m 0755 etcd-v3.6.6-linux-amd64/etcdutl /usr/local/bin/etcdutl
```

`install` copies and sets permissions in one step. `0755` = `rwxr-xr-x`.

### 10.4 — Verify

```bash
etcdctl version
etcdutl version
```

Expected:

```
etcdctl version: 3.6.6
API version: 3.6

etcdutl version: 3.6.6
API version: 3.6
```

### Why two tools?

| Tool | Purpose | Talks to |
|------|---------|----------|
| `etcdctl` | Online operations | Running etcd (via network) |
| `etcdutl` | Offline operations | Snapshot files (via disk) |

---

# Part 5 — Creating and Verifying the Snapshot

## 11. Step 1 — Validate the etcd Cluster

Before taking a backup, **prove the cluster is healthy**.

### 11.1 — List etcd members

```bash
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table
```

Expected output:

```
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |       NAME        |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
| 574699e63127f8c0 | started | cp03.faekcorp.lab | https://10.70.41.250:2380 | https://10.70.41.250:2379 |      false |
| 6108a621ed6af19b | started | cp01.faekcorp.lab |   https://10.70.21.6:2380 |   https://10.70.21.6:2379 |      false |
| ee6b7ea6225a4e3e | started | cp02.faekcorp.lab | https://10.70.31.209:2380 | https://10.70.31.209:2379 |      false |
+------------------+---------+-------------------+---------------------------+---------------------------+------------+
```

**What to verify:**

- All three members present
- All `STATUS = started`
- Names match your CP hostnames
- `IS LEARNER = false` for all

### 11.2 — Check endpoint health

```bash
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health -w table
```

Expected:

```
+------------------------+--------+------------+-------+
|        ENDPOINT        | HEALTH |    TOOK    | ERROR |
+------------------------+--------+------------+-------+
| https://127.0.0.1:2379 |   true | 6.115307ms |       |
+------------------------+--------+------------+-------+
```

### 11.3 — Understanding the flags

Every flag is required:

| Flag | Meaning |
|------|---------|
| `sudo` | Cert files are root-readable only |
| `--endpoints` | Which etcd to talk to (`127.0.0.1:2379` = local) |
| `--cacert` | Trust this CA to verify etcd's server cert |
| `--cert` | Our client cert (mTLS) |
| `--key` | Our private key (mTLS) |

### Optional — shell alias for convenience

```bash
cat >> ~/.bashrc << 'EOF'
alias etcdctl-local='sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'
EOF
source ~/.bashrc
```

Then:

```bash
etcdctl-local member list -w table
etcdctl-local endpoint health
```

## 12. Step 2 — Take the Snapshot

### 12.1 — The command

```bash
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot.db
```

### 12.2 — What happens underneath

1. etcdctl connects to etcd on cp01
2. etcd coordinates a **consistent** read of its state through Raft
3. The snapshot is streamed back to etcdctl
4. etcdctl writes to `snapshot.db.part` first (atomic write pattern)
5. On success, renames `.part` → `snapshot.db`

### 12.3 — Expected output

```
{"level":"info","ts":"...","msg":"created temporary db file","path":"/var/lib/etcd/snapshot.db.part"}
{"level":"info","ts":"...","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"...","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"...","msg":"completed snapshot read; closing"}
{"level":"info","ts":"...","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"12 MB","took":"209.548061ms","etcd-version":"3.6.0"}
{"level":"info","ts":"...","msg":"saved","path":"/var/lib/etcd/snapshot.db"}
Snapshot saved at /var/lib/etcd/snapshot.db
Server version 3.6.0
```

**Line by line:**

| Line | Meaning |
|------|---------|
| `created temporary db file` | Writing to `.part` first |
| `opened snapshot stream` | Stream established |
| `fetching snapshot` | Downloading data |
| `completed snapshot read` | Download complete |
| `fetched snapshot` (size) | Data received |
| `saved` | Renamed `.part` → final file |

Total time: ~210 ms. **No downtime, no disruption.**

### 12.4 — Why `.part` and rename?

Atomic write pattern. If the process is killed mid-write:

- Without `.part`: you might have a half-written `snapshot.db` that looks complete
- With `.part`: you either have the full `snapshot.db` or only `.part`

## 13. Step 3 — Verify File Presence

### 13.1 — Check the file exists

```bash
sudo ls -lh /var/lib/etcd/snapshot.db
```

Expected:

```
-rw------- 1 root root 12M Oct  6 21:11 /var/lib/etcd/snapshot.db
```

### 13.2 — Understanding the permissions

```
-rw-------  1  root  root  12M  ...
│││││││││  │   │     │     │
│││││││││  │   │     │     └─ 12 MB size
│││││││││  │   │     └─ group: root
│││││││││  │   └─ owner: root
│││││││││  └─ link count
││││││││└─ others: none
│││││││└─ group: none
││││││└─ owner: read + write
│││││└─ regular file
```

Permissions: **600** (owner read/write, nothing else).

**Why 600?** Because a snapshot contains **every Kubernetes Secret**. If any unprivileged user could read this file, they'd have full access to every secret.

**Rule:** never `chmod 644` an etcd snapshot. Never commit it to git. Never email it.

## 14. Step 4 — Verify Snapshot Integrity

### 14.1 — The command

```bash
sudo etcdutl snapshot status /var/lib/etcd/snapshot.db -w table
```

**Note:** this is `etcdutl`, not `etcdctl`.

| Tool | What it does |
|------|--------------|
| `etcdctl snapshot status` | Talks to running etcd (deprecated) |
| `etcdutl snapshot status` | Reads the file directly (offline) |

### 14.2 — Expected output

```
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| fc420e4c |   214775 |        387 |      12 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

### 14.3 — Field-by-field

| Column | Meaning | What to watch |
|--------|---------|---------------|
| **HASH** | Integrity fingerprint | Compare across copies — must match |
| **REVISION** | etcd global revision at snapshot time | Increases monotonically |
| **TOTAL KEYS** | Number of key-value pairs | Sanity check |
| **TOTAL SIZE** | Logical data size | May differ from file size |
| **VERSION** | etcd version that created it | Must be compatible for restore |

### 14.4 — The "verified vs restorable" distinction

```
Level 1 — Present      ✓  (file exists)
Level 2 — Verified     ✓  (etcdutl snapshot status succeeds)
Level 3 — Restorable   ❓  (would need a full restore test)
```

**Level 3 is the real test.** A snapshot that passes verification but fails to restore would leave you in disaster recovery with no working backup.

**Recommendation:** perform a restore test on an isolated cluster after this runbook. A backup that has never been successfully restored is only an assumption.

## 15. Step 5 — Copy Off the Node

### 15.1 — Why this step exists

The snapshot at `/var/lib/etcd/snapshot.db` is on the **same disk** as the live etcd data. If that disk fails, both are gone.

```
cp01
  │
  ├── /var/lib/etcd/member/     ← live data
  └── /var/lib/etcd/snapshot.db ← backup

Disk fails → BOTH lost
```

**This is not a backup.** It's a copy.

### 15.2 — The copy command with timestamp

```bash
sudo cp /var/lib/etcd/snapshot.db \
  /home/ubuntu/etcd-snapshot-cp01-$(date +%Y%m%d-%H%M%S).db
```

Filename breakdown:

```
etcd-snapshot-cp01-20261006-211105.db
      │         │      │
      │         │      └─ timestamp
      │         └─ source node
      └─ what it is
```

Why timestamped:
- Each snapshot is a unique file
- Sorted names = chronological order
- No accidental overwrites

### 15.3 — Fix ownership and permissions

```bash
sudo chown ubuntu:ubuntu /home/ubuntu/etcd-snapshot-cp01-*.db
sudo chmod 600 /home/ubuntu/etcd-snapshot-cp01-*.db
```

### 15.4 — Verify the copy

```bash
ls -lh /home/ubuntu/etcd-snapshot-cp01-*.db
sudo etcdutl snapshot status /home/ubuntu/etcd-snapshot-cp01-*.db -w table
```

The **HASH must match** the original:

```
| fc420e4c | ...    ← same as /var/lib/etcd/snapshot.db
```

### 15.5 — Optional: cryptographic verification

```bash
sudo sha256sum /var/lib/etcd/snapshot.db /home/ubuntu/etcd-snapshot-cp01-*.db
```

Both SHA-256 hashes must match.

## 16. Snapshot Naming and Retention

### 16.1 — Naming conventions

**Can you use any name?** Yes. `etcdctl snapshot save <path>` writes the file to whatever path you give it.

```bash
# All valid
etcdctl snapshot save /var/lib/etcd/snapshot.db
etcdctl snapshot save /var/lib/etcd/etcd-snapshot-20261006.db
etcdctl snapshot save /home/ubuntu/backups/etcd.db
etcdctl snapshot save /tmp/arbitrary-name
```

**Requirements:**

- Directory must exist
- You must have write permission
- Filename must not already exist (or it gets overwritten)
- Enough disk space

### 16.2 — Overwrite vs separate files

There is **no automatic versioning** in etcdctl.

```bash
# First run
etcdctl snapshot save /var/lib/etcd/snapshot.db      → creates the file

# Second run (same filename)
etcdctl snapshot save /var/lib/etcd/snapshot.db      → OVERWRITES
```

**Risk:** if you use the same filename and take a snapshot during a bad cluster state, you've destroyed your last good backup.

**Solution:** always timestamp filenames.

### 16.3 — Recommended format

```
etcd-snapshot-<cluster>-<host>-YYYYMMDD-HHMMSS.db
```

Example:

```
etcd-snapshot-faekcorp-lab-cp01-20261006-211105.db
etcd-snapshot-faekcorp-lab-cp01-20261007-030000.db
etcd-snapshot-faekcorp-lab-cp01-20261007-090000.db
```

### 16.4 — S3 key structure

```
s3://ha-cluster-s3-etcd-lab/
  └── etcd-snapshots/
      └── etcd-snapshot-cp01-20261006-211105.db
```

Or hierarchical by date:

```
s3://ha-cluster-s3-etcd-lab/etcd-snapshots/2026/10/06/etcd-snapshot-...
```

---

# Part 6 — Uploading to S3 with IAM Role

## 17. Why IAM Roles, Not Static Keys

| Aspect | Static keys | IAM role |
|--------|-------------|----------|
| Rotation | Manual | Automatic (hourly) |
| On disk | Yes (`~/.aws/credentials`) | No — in memory only |
| Valid off-instance | Yes (anywhere) | No (bound to instance) |
| Leak impact | Forever, from anywhere | ≤1 hour, from cp01 only |
| Audit trail | Basic | Full (instance ID in CloudTrail) |

**IAM roles are the production-grade choice.**

## 18. The Two Identities: You vs the Instance

There are **two separate AWS identities** involved. They are not the same.

### Identity 1 — YOU (the human admin)

```
    Your laptop
         │
         │  aws configure (with YOUR personal access keys)
         │  OR
         │  aws sso login
         │
         ▼
    AWS account (admin permissions)
         │
         │  Create IAM policy
         │  Create IAM role
         │  Attach role to cp01
         ▼
    Done — role exists and is attached
```

**Who:** you, on your laptop.
**Purpose:** make changes to the AWS account.
**Credentials:** your personal credentials, admin-level, used sparingly.

### Identity 2 — cp01 (the instance)

```
    cp01 (EC2 instance)
         │
         │  IAM role attached → IMDS provides temporary credentials
         │  (no ~/.aws/credentials file)
         │
         ▼
    AWS account (limited to backup permissions)
         │
         │  aws s3 cp → uploads snapshot
         ▼
    S3 bucket
```

**Who:** the EC2 instance, wearing the `cp01-etcd-backup-role`.
**Purpose:** perform the specific task (upload to S3).
**Credentials:** automatic, temporary, provided by IMDS.

## 19. What Goes Where

| Thing | Where it goes | Why |
|-------|--------------|-----|
| **IAM policy** | AWS account | Defines what actions are allowed |
| **IAM role** | AWS account | Identity that holds the policy |
| **Attach role to cp01** | AWS account → cp01 | Gives cp01 permission to use the role |
| **AWS CLI** | cp01 (install) | The tool that makes the API calls |
| **Access keys (yours)** | Your laptop only | For you to manage AWS as an admin |
| **`aws configure`** | Your laptop only | Sets up your personal CLI |
| **`~/.aws/credentials` on cp01** | **Nothing** | Should NOT exist — role provides credentials |

**One sentence:** your laptop sets up the role; the role authorizes cp01; cp01 never sees a static key.

## 20. Console Setup Steps

There are three things to create or attach in the AWS Console.

### 20.1 — Create the IAM Policy

1. Sign in to the AWS Console, open **IAM**.
2. Left navigation: **Policies** → **Create policy**.
3. Switch to the **JSON** tab.
4. Paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EtcdSnapshotUpload",
            "Effect": "Allow",
            "Action": ["s3:PutObject", "s3:PutObjectAcl"],
            "Resource": "arn:aws:s3:::ha-cluster-s3-etcd-lab/etcd-snapshots/*"
        },
        {
            "Sid": "EtcdSnapshotList",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::ha-cluster-s3-etcd-lab"
        },
        {
            "Sid": "EtcdSnapshotRead",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::ha-cluster-s3-etcd-lab/etcd-snapshots/*"
        }
    ]
}
```

5. **Next**.
6. Name: `cp01-etcd-backup-upload-policy`.
7. **Create policy**.

### 20.2 — Create the IAM Role

1. In IAM, left navigation: **Roles** → **Create role**.
2. Trusted entity: **AWS service**.
3. Use case: **EC2** → **Next**.
4. Search for your policy, check the box → **Next**.
5. Name: `cp01-etcd-backup-role`.
6. **Create role**.

> An **instance profile** with the same name is created automatically.

### 20.3 — Attach the Role to cp01

1. Open **EC2**.
2. Left navigation: **Instances**.
3. Check the box next to cp01.
4. **Actions** → **Security** → **Modify IAM role**.
5. Select `cp01-etcd-backup-role`.
6. **Update IAM role**.

No restart needed. The AWS CLI on cp01 picks up the role on the next command.

## 21. CLI Alternative

If you prefer the CLI (from your laptop with admin credentials):

```bash
# 1. Create the policy
aws iam create-policy \
  --policy-name cp01-etcd-backup-upload-policy \
  --policy-document file://policy.json

# 2. Create the role (trust policy allows EC2 to assume it)
aws iam create-role \
  --role-name cp01-etcd-backup-role \
  --assume-role-policy-document file://trust-policy.json

# 3. Attach the policy to the role
aws iam attach-role-policy \
  --role-name cp01-etcd-backup-role \
  --policy-arn arn:aws:iam::123456789012:policy/cp01-etcd-backup-upload-policy

# 4. Attach role to the EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-0123456789abcdef0 \
  --iam-instance-profile Name=cp01-etcd-backup-role
```

Where `trust-policy.json` is:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {"Service": "ec2.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## 22. Installing AWS CLI on cp01 (No Configuration Needed)

### 22.1 — Install

```bash
cd /tmp
sudo apt update
sudo apt install -y unzip curl

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

aws --version
```

Expected:

```
aws-cli/2.x.x Python/3.x.x Linux/... 
```

### 22.2 — Do NOT run `aws configure` on cp01

Running `aws configure` on cp01 would write static credentials to `~/.aws/credentials`. This is exactly what we're avoiding.

The AWS CLI's credential resolution order:

```
1. Command-line args
2. Environment variables
3. ~/.aws/credentials file
4. ~/.aws/config file
5. Container credentials
6. EC2 Instance Metadata Service (IMDS) ← we want this
```

If steps 1-5 are empty, the CLI falls through to IMDS and picks up the IAM role. So we install but do not configure.

### 22.3 — Verify the installation found the role

```bash
aws sts get-caller-identity
```

Expected:

```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:i-0123456789abcdef0",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/cp01-etcd-backup-role/i-0123456789abcdef0"
}
```

The `assumed-role` prefix confirms the IAM role is providing credentials automatically.

### 22.4 — If it fails

| Error | Fix |
|-------|-----|
| `Unable to locate credentials` | Attach role to cp01 |
| `Access Denied` on `s3 ls` | Check IAM policy |
| `NoSuchBucket` | Bucket name typo, wrong region |

**Debug IMDS reachability:**

```bash
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

If this returns a role name, IMDS is fine.

### 22.5 — Check no static credentials lurk

```bash
# 1. No credentials file
ls -la ~/.aws/ 2>/dev/null || echo "no ~/.aws directory — good"

# 2. No AWS env vars
env | grep -i aws || echo "no AWS env vars — good"
```

Both should report "good." If not, remove the credentials file and unset the env vars.

## 23. Verifying the Role Works

Run on **cp01**:

```bash
# 1. AWS CLI is installed
aws --version

# 2. IAM role provides credentials
aws sts get-caller-identity

# 3. S3 access works
aws s3 ls s3://ha-cluster-s3-etcd-lab/
```

| Result | Meaning |
|--------|---------|
| All three work | Ready to upload |
| `Unable to locate credentials` | Role not attached or IMDS disabled |
| `Access Denied` | IAM policy missing permissions |
| `NoSuchBucket` | Bucket name wrong, wrong region |

## 24. The Upload

### 24.1 — The command

```bash
aws s3 cp /home/ubuntu/etcd-snapshot-cp01-*.db \
  s3://ha-cluster-s3-etcd-lab/etcd-snapshots/ \
  --sse AES256
```

**`--sse AES256`** = server-side encryption with AES-256. S3 manages the key. Mandatory for snapshots containing secrets.

### 24.2 — Expected output

```
upload: ./etcd-snapshot-cp01-20261006-213045.db to s3://ha-cluster-s3-etcd-lab/etcd-snapshots/etcd-snapshot-cp01-20261006-213045.db
```

### 24.3 — Verify the upload

```bash
aws s3 ls s3://ha-cluster-s3-etcd-lab/etcd-snapshots/
```

Expected:

```
2026-10-06 21:35:12   12582912 etcd-snapshot-cp01-20261006-213045.db
```

Size should match your local file.

### 24.4 — Strong verification (MD5/ETag)

```bash
# MD5 of local file
md5sum /home/ubuntu/etcd-snapshot-cp01-*.db

# ETag of S3 object
aws s3api head-object \
  --bucket ha-cluster-s3-etcd-lab \
  --key etcd-snapshots/$(basename /home/ubuntu/etcd-snapshot-cp01-*.db) \
  --query ETag --output text
```

The S3 ETag (in hex, without quotes) should match the local MD5.

### 24.5 — Cleanup

After verifying the S3 upload:

```bash
# Remove the local snapshot (only after S3 is verified)
sudo rm /var/lib/etcd/snapshot.db
rm /home/ubuntu/etcd-snapshot-cp01-*.db
```

**Why clean up?** In a scheduled backup, local snapshots accumulate. Disk fills. etcd crashes.

---

# Part 7 — Production Considerations

## 25. Automation and Scheduling

**Do NOT use a Kubernetes CronJob for etcd backups.**

If the cluster is broken, a CronJob inside the cluster can't run. The backup mechanism must not depend on the system it's backing up.

**Use instead:**

- systemd timer on a CP node
- External scheduler (Jenkins, GitHub Actions, dedicated backup server)
- Cloud-native tools (Velero, Kasten)

### Full automation flow

```
Every 6 hours:

  1. SSH to cp01
  2. Run etcdctl snapshot save /tmp/etcd-snapshot-<timestamp>.db
  3. Run etcdutl snapshot status (verify integrity)
  4. If verification fails → alert, abort
  5. aws s3 cp /tmp/snapshot.db s3://bucket/etcd-snapshots/
  6. Verify S3 upload (MD5/ETag match)
  7. Delete local /tmp/snapshot.db
  8. Log success/failure to monitoring
```

## 26. Least-Privilege IAM Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EtcdSnapshotUpload",
            "Effect": "Allow",
            "Action": ["s3:PutObject", "s3:PutObjectAcl"],
            "Resource": "arn:aws:s3:::ha-cluster-s3-etcd-lab/etcd-snapshots/*"
        },
        {
            "Sid": "EtcdSnapshotList",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::ha-cluster-s3-etcd-lab"
        },
        {
            "Sid": "EtcdSnapshotRead",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::ha-cluster-s3-etcd-lab/etcd-snapshots/*"
        }
    ]
}
```

**Notably absent:** `s3:DeleteObject`, `s3:DeleteBucket`, `s3:*` wildcards. The role cannot delete backups.

## 27. Retention and Lifecycle

### Recommended retention

| Age | Retention | Why |
|-----|-----------|-----|
| 0–24h | Hourly snapshots | Immediate recovery |
| 1–30d | Daily snapshots | Recent disaster recovery |
| 1–12mo | Weekly/monthly | Long-term audit, compliance |

### S3 lifecycle policy

```
Rule: etcd-snapshots
  Transition to S3 Standard-IA   after 7 days
  Transition to Glacier Instant  after 30 days
  Delete                         after 365 days
```

## 28. Monitoring and Alerting

- Alert if backup hasn't run in >12 hours
- Alert if `etcdutl snapshot status` fails
- Alert if S3 upload fails
- Alert if snapshot size drops suddenly (possible data loss)
- Alert on unexpected S3 access (potential breach)

## 29. What's Not Covered

- **Restore procedure** — separate runbook, done on isolated test cluster
- **Volume backups** (PersistentVolumes, databases) — separate tools (Velero, app-specific)
- **Certificate backups** (etcd PKI) — separate procedure
- **Multi-cluster aggregation** — different S3 key strategies

**A backup that has never been restored is only an assumption.**

---

# Part 8 — Reference

## 30. Complete Command Reference

### etcdctl / etcdutl

| Command | Purpose |
|---------|---------|
| `etcdctl version` | Verify etcdctl installed |
| `etcdutl version` | Verify etcdutl installed |
| `etcdctl member list` | List etcd members |
| `etcdctl endpoint health` | Check etcd health |
| `etcdctl snapshot save <path>` | Create backup snapshot |
| `etcdutl snapshot status <path>` | Verify snapshot integrity |
| `etcdutl snapshot restore <path>` | Restore (separate runbook) |

### AWS CLI

| Command | Purpose |
|---------|---------|
| `aws --version` | Verify AWS CLI installed |
| `aws sts get-caller-identity` | Verify AWS identity |
| `aws s3 ls <bucket>/` | List S3 objects |
| `aws s3 cp <local> <s3>` | Upload to S3 |
| `aws s3api head-object` | Get S3 object metadata (ETag) |
| `aws iam create-policy` | Create IAM policy |
| `aws iam create-role` | Create IAM role |
| `aws iam attach-role-policy` | Attach policy to role |
| `aws ec2 associate-iam-instance-profile` | Attach role to instance |

### Local file utilities

| Command | Purpose |
|---------|---------|
| `sha256sum <file>` | SHA-256 hash |
| `md5sum <file>` | MD5 hash |

## 31. Key Paths

| Path | What it is |
|------|-----------|
| `/etc/kubernetes/manifests/etcd.yaml` | etcd static pod manifest |
| `/etc/kubernetes/pki/etcd/ca.crt` | etcd cluster CA cert |
| `/etc/kubernetes/pki/etcd/server.crt` | etcd server cert |
| `/etc/kubernetes/pki/etcd/server.key` | etcd server private key |
| `/etc/kubernetes/pki/etcd/peer.crt` | etcd peer cert (not used by etcdctl) |
| `/var/lib/etcd/` | etcd data directory |
| `/var/lib/etcd/member/snap/db` | Live key-value database |
| `/var/lib/etcd/member/wal/` | Write-ahead log |
| `/var/lib/etcd/snapshot.db` | Backup snapshot (created by us) |
| `/usr/local/bin/etcdctl` | etcdctl binary |
| `/usr/local/bin/etcdutl` | etcdutl binary |
| `~/.aws/credentials` | Static AWS credentials (should NOT exist on cp01) |

## 32. Summary — The Whole Flow at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│  Install     │  etcdctl + etcdutl 3.6.6 on cp01               │
├──────────────────────────────────────────────────────────────┤
│  Validate    │  member list + endpoint health                  │
├──────────────────────────────────────────────────────────────┤
│  Snapshot    │  etcdctl snapshot save /var/lib/etcd/snapshot.db│
├──────────────────────────────────────────────────────────────┤
│  Verify      │  etcdutl snapshot status (HASH, revision, keys) │
├──────────────────────────────────────────────────────────────┤
│  Copy        │  home directory with timestamped name           │
├──────────────────────────────────────────────────────────────┤
│  AWS setup   │  IAM policy + role + attach to cp01             │
│              │  (done from your laptop, not cp01)              │
├──────────────────────────────────────────────────────────────┤
│  Install CLI │  aws CLI on cp01, no aws configure              │
├──────────────────────────────────────────────────────────────┤
│  Verify role │  aws sts get-caller-identity                    │
├──────────────────────────────────────────────────────────────┤
│  Upload      │  aws s3 cp with --sse AES256                    │
├──────────────────────────────────────────────────────────────┤
│  Verify S3   │  s3 ls + ETag/MD5 comparison                    │
├──────────────────────────────────────────────────────────────┤
│  Cleanup     │  delete local copies                            │
└──────────────────────────────────────────────────────────────┘
```

### The key mental model

```
             KUBERNETES CLUSTER
                     │
              kube-apiserver
                     │
                     ▼
                   ETCD
                     │
          ┌──────────┴──────────┐
          │                     │
      live storage          backup snapshot
      /var/lib/etcd/        snapshot.db
      /member/                   │
                                 │  upload (via IAM role)
                                 ▼
                              S3 bucket
                              (encrypted, versioned)
                                 │
                                 │  restore (when needed)
                                 ▼
                              new etcd
```

**The backup is the S3 object. Everything else is plumbing to get it there.**

---
