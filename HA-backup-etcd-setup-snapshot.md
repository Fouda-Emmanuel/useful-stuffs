# Kubernetes etcd Backup Setup — Complete Runbook

A hands-on, step-by-step guide to backing up an etcd cluster on a kubeadm-based Kubernetes cluster. Written from real experiments on a 3-control-plane HA cluster (Kubernetes 1.35.8, etcd 3.6.6).

---

## Table of Contents

1. [Why etcd Backup Matters](#1-why-etcd-backup-matters)
2. [The Architecture We're Working With](#2-the-architecture-were-working-with)
3. [Where to Install etcdctl / etcdutl](#3-where-to-install-etcdctl--etcdutl)
4. [Understanding `/var/lib/etcd/`](#4-understanding-varlibetcd)
5. [Step 1 — Install etcdctl and etcdutl](#5-step-1--install-etcdctl-and-etcdutl)
6. [Step 2 — Validate the etcd Cluster](#6-step-2--validate-the-etcd-cluster)
7. [Step 3 — Create the Snapshot](#7-step-3--create-the-snapshot)
8. [Step 4 — Verify the Snapshot File](#8-step-4--verify-the-snapshot-file)
9. [Step 5 — Verify Snapshot Integrity](#9-step-5--verify-snapshot-integrity)
10. [Step 6 — Copy the Snapshot Off the Node](#10-step-6--copy-the-snapshot-off-the-node)
11. [Upload to S3 with IAM Role](#11-upload-to-s3-with-iam-role)
12. [Snapshot Naming and Retention](#12-snapshot-naming-and-retention)
13. [Production Considerations](#13-production-considerations)
14. [Complete Command Reference](#14-complete-command-reference)

---

## 1. Why etcd Backup Matters

### The mental model

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

### HA is not backup

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

---

## 2. The Architecture We're Working With

Our cluster:

```
                    ┌──────────────────┐
                    │     Bastion      │
                    │  kubectl / aws   │
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

---

## 3. Where to Install etcdctl / etcdutl

### The decision

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

Neither is necessary. On cp01, everything is already in the right place:

```
cp01
  │
  ├── etcd (running)
  ├── /etc/kubernetes/pki/etcd/  (certs)
  └── /usr/local/bin/etcdctl      ← we'll add this
       /usr/local/bin/etcdutl     ← and this
```

**Golden rule:** keep admin tooling as close to the resource as possible, and minimize credential distribution.

---

## 4. Understanding `/var/lib/etcd/`

Before touching anything, understand what you're looking at. This directory is one etcd member's local persistent storage.

### The tree

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

### What each piece means

| Path | What it is | Managed by |
|------|-----------|------------|
| `member/` | This member's persistent storage | etcd |
| `snap/` | Snapshot/database storage area | etcd |
| `snap/db` | **Live key-value database** (bbolt format) | etcd |
| `snap/*.snap` | Internal checkpoints (metadata only, ~9 KB each) | etcd |
| `wal/` | Write-ahead log (Raft log entries) | etcd |
| `wal/*.wal` | Individual WAL segments (fixed max size) | etcd |

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

### Three different "snapshots" — don't confuse them

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

## 5. Step 1 — Install etcdctl and etcdutl

### 5.1 — Determine the running etcd version

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

### 5.2 — Download and extract on cp01

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

### 5.3 — Install the two tools

```bash
sudo install -m 0755 etcd-v3.6.6-linux-amd64/etcdctl /usr/local/bin/etcdctl
sudo install -m 0755 etcd-v3.6.6-linux-amd64/etcdutl /usr/local/bin/etcdutl
```

`install` copies and sets permissions in one step. `0755` = `rwxr-xr-x`.

### 5.4 — Verify

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

`etcdctl snapshot save` requires a running etcd. `etcdutl snapshot status` reads a snapshot file directly.

---

## 6. Step 2 — Validate the etcd Cluster

Before taking a backup, **prove the cluster is healthy**. Otherwise you might snapshot a broken state.

### 6.1 — List etcd members

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

### 6.2 — Check endpoint health

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

**What to verify:**
- `HEALTH = true`
- Latency in single-digit milliseconds (or low tens)
- No error string

### 6.3 — Understanding the flags

Every flag is required:

| Flag | Meaning |
|------|---------|
| `sudo` | Cert files are root-readable only |
| `--endpoints` | Which etcd to talk to (`127.0.0.1:2379` = local) |
| `--cacert` | Trust this CA to verify etcd's server cert |
| `--cert` | Our client cert (mTLS) |
| `--key` | Our private key (mTLS) |

### Optional: shell alias for convenience

Because typing all flags every time is painful:

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

---

## 7. Step 3 — Create the Snapshot

### 7.1 — The command

```bash
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot.db
```

### 7.2 — What happens underneath

1. etcdctl connects to etcd on cp01
2. etcd coordinates a **consistent** read of its state through Raft
3. The snapshot is streamed back to etcdctl
4. etcdctl writes to `snapshot.db.part` first (atomic write pattern)
5. On success, renames `.part` → `snapshot.db`

### 7.3 — Expected output

```
{"level":"info","ts":"2026-10-06T21:11:04.837502Z","caller":"snapshot/v3_snapshot.go:83","msg":"created temporary db file","path":"/var/lib/etcd/snapshot.db.part"}
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

### 7.4 — Why `.part` and rename?

Atomic write pattern. If the process is killed mid-write:

- Without `.part`: you might have a half-written `snapshot.db` that looks complete
- With `.part`: you either have the full `snapshot.db` or only `.part`

You can check:

```bash
ls -l /var/lib/etcd/
```

If you see `.part` and no `.db`, the snapshot failed. If you see `.db`, it succeeded.

---

## 8. Step 4 — Verify the Snapshot File

### 8.1 — Check the file exists

```bash
sudo ls -lh /var/lib/etcd/snapshot.db
```

Expected:

```
-rw------- 1 root root 12M Oct  6 21:11 /var/lib/etcd/snapshot.db
```

### 8.2 — Understanding the permissions

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

**Why 600?** Because a snapshot contains **every Kubernetes Secret** — database passwords, service account tokens, TLS private keys. If any unprivileged user could read this file, they'd have full access to every secret.

**Rule:** never `chmod 644` an etcd snapshot. Never commit it to git. Never email it.

### 8.3 — The `file` command

```bash
sudo file /var/lib/etcd/snapshot.db
```

Output:

```
/var/lib/etcd/snapshot.db: data
```

Just "data" — because it's a binary format `file` doesn't recognize. This is expected. `etcdutl` is the only tool that knows how to read it.

---

## 9. Step 5 — Verify Snapshot Integrity

### 9.1 — Why verification matters

A file on disk is not automatically a valid snapshot. It could be:

- Truncated (write interrupted)
- Corrupted (bad disk, silent bit flip)
- Incompatible (different etcd version)

Verifying **now** means you catch problems **before** you need the backup in an emergency.

### 9.2 — The command

```bash
sudo etcdutl snapshot status /var/lib/etcd/snapshot.db -w table
```

**Note:** this is `etcdutl`, not `etcdctl`. Different tool.

| Tool | What it does |
|------|--------------|
| `etcdctl snapshot status` | Talks to running etcd (deprecated) |
| `etcdutl snapshot status` | Reads the file directly (offline) |

The `etcdutl` version works even when etcd is down — which is exactly the situation during disaster recovery.

### 9.3 — Expected output

```
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| fc420e4c |   214775 |        387 |      12 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

### 9.4 — Field-by-field

| Column | Meaning | What to watch |
|--------|---------|---------------|
| **HASH** | Integrity fingerprint | Compare across copies — must match |
| **REVISION** | etcd global revision at snapshot time | Increases monotonically; higher = more recent |
| **TOTAL KEYS** | Number of key-value pairs | Sanity check — should match expected object count |
| **TOTAL SIZE** | Logical data size | May differ from file size |
| **VERSION** | etcd version that created it | Must be compatible with target for restore |

### 9.5 — What this proves

✅ The file is readable (not truncated)
✅ Internal structure is valid
✅ Data is consistent (HASH computed successfully)
✅ Version is compatible

### 9.6 — The "verified vs restorable" distinction

```
Level 1 — Present      ✓  (file exists)
Level 2 — Verified     ✓  (etcdutl snapshot status succeeds)
Level 3 — Restorable   ❓  (would need a full restore test)
```

**Level 3 is the real test.** A snapshot that passes verification but fails to restore would leave you in disaster recovery with no working backup.

**Recommendation:** after this runbook, perform a restore test on an isolated cluster. A backup that has never been successfully restored is only an assumption.

---

## 10. Step 6 — Copy the Snapshot Off the Node

### 10.1 — Why this step exists

The snapshot at `/var/lib/etcd/snapshot.db` is on the **same disk** as the live etcd data. If that disk fails, both are gone.

```
cp01
  │
  ├── /var/lib/etcd/member/     ← live data
  └── /var/lib/etcd/snapshot.db ← backup

Disk fails → BOTH lost
```

**This is not a backup.** It's a copy.

### 10.2 — The 3-2-1 rule

A real backup strategy follows:

- **3** copies of data
- **2** different media / locations
- **1** offsite

For etcd snapshots, that usually means:

```
1. Snapshot on cp01 (temporary)
2. Copy to home directory (convenient)
3. Upload to S3 (durable, offsite)
```

### 10.3 — The copy command with timestamp

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

### 10.4 — Fix ownership and permissions

```bash
sudo chown ubuntu:ubuntu /home/ubuntu/etcd-snapshot-cp01-*.db
sudo chmod 600 /home/ubuntu/etcd-snapshot-cp01-*.db
```

`cp` preserves the source's permissions (600 root:root), so we:
- Change ownership to your user (so you can `scp`, `aws s3 cp`, etc.)
- Re-apply 600 to make sure permissions are still tight

### 10.5 — Verify the copy

```bash
ls -lh /home/ubuntu/etcd-snapshot-cp01-*.db
sudo etcdutl snapshot status /home/ubuntu/etcd-snapshot-cp01-*.db -w table
```

The **HASH must match** the original:

```
| fc420e4c | ...    ← same as /var/lib/etcd/snapshot.db
```

Same HASH = same content. The copy is not corrupted.

### 10.6 — Optional: cryptographic verification

```bash
sudo sha256sum /var/lib/etcd/snapshot.db /home/ubuntu/etcd-snapshot-cp01-*.db
```

Both SHA-256 hashes must match. This is stronger than the etcdutl HASH (which is over logical content, not the file).

---

## 11. Upload to S3 with IAM Role

### 11.1 — Why IAM roles, not static keys

| Aspect | Static keys | IAM role |
|--------|-------------|----------|
| Rotation | Manual | Automatic (hourly) |
| On disk | Yes (`~/.aws/credentials`) | No — in memory only |
| Valid off-instance | Yes (anywhere) | No (bound to instance) |
| Leak impact | Forever, from anywhere | ≤1 hour, from cp01 only |
| Audit trail | Basic | Full (instance ID in CloudTrail) |

IAM roles are the production-grade choice.

### 11.2 — Verify IAM role is attached

On cp01:

```bash
aws --version
aws sts get-caller-identity
```

Expected ARN:

```json
{
    "Arn": "arn:aws:sts::123456789012:assumed-role/cp01-etcd-backup-role/i-0123456789abcdef0"
}
```

The `assumed-role` prefix is the signature of a role-based identity.

### 11.3 — Verify S3 access

```bash
aws s3 ls s3://ha-cluster-s3-etcd-lab/
```

Empty output = bucket exists, no objects (or you lack `ListBucket` — but the policy should include it).

### 11.4 — Upload with encryption

```bash
aws s3 cp /home/ubuntu/etcd-snapshot-cp01-*.db \
  s3://ha-cluster-s3-etcd-lab/etcd-snapshots/ \
  --sse AES256
```

**`--sse AES256`** = Server-side encryption with AES-256. S3 manages the key. The snapshot contains every Kubernetes Secret, so encryption at rest is mandatory.

### 11.5 — Verify the upload

```bash
aws s3 ls s3://ha-cluster-s3-etcd-lab/etcd-snapshots/
```

Expected:

```
2026-10-06 21:35:12   12582912 etcd-snapshot-cp01-20261006-213045.db
```

Size should match your local file.

### 11.6 — Verify integrity (the strong check)

```bash
# MD5 of the local file
md5sum /home/ubuntu/etcd-snapshot-cp01-*.db

# ETag of the S3 object
aws s3api head-object \
  --bucket ha-cluster-s3-etcd-lab \
  --key etcd-snapshots/etcd-snapshot-cp01-20261006-213045.db \
  --query ETag --output text
```

The S3 ETag (in hex, without quotes) should match the local MD5. If yes, the file uploaded byte-perfect.

### 11.7 — Optional: upload from the console

If you prefer the AWS Console:

1. S3 → `ha-cluster-s3-etcd-lab` → **Create folder** → `etcd-snapshots`
2. Click into the folder → **Upload**
3. **Add files** → select the snapshot from your local machine
4. Expand **Properties** → **Server-side encryption settings** → **Amazon S3 managed key (SSE-S3)**
5. Click **Upload**

The console method requires the file on your local machine first (scp it from cp01).

### 11.8 — Cleanup after upload

After verifying the S3 upload:

```bash
# Remove the local snapshot (only after S3 is verified)
sudo rm /var/lib/etcd/snapshot.db
rm /home/ubuntu/etcd-snapshot-cp01-*.db
```

**Why clean up?** In a scheduled backup, local snapshots accumulate. Disk fills. etcd crashes. Very bad day.

**But** — for a lab, you may want to keep the local copy until you've tested a restore. Your call.

---

## 12. Snapshot Naming and Retention

### 12.1 — Naming conventions

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

**The command ignores the `.db` extension** — it's convention, not requirement.

### 12.2 — Overwrite vs separate files

There is **no automatic versioning** in etcdctl.

```bash
# First run
etcdctl snapshot save /var/lib/etcd/snapshot.db      → creates the file

# Second run (same filename)
etcdctl snapshot save /var/lib/etcd/snapshot.db      → OVERWRITES
```

**Risk:** if you use the same filename and take a snapshot during a bad cluster state, you've destroyed your last good backup.

**Solution:** always timestamp filenames.

### 12.3 — Recommended format

```
etcd-snapshot-<cluster>-<host>-YYYYMMDD-HHMMSS.db
```

Example:

```
etcd-snapshot-faekcorp-lab-cp01-20261006-211105.db
etcd-snapshot-faekcorp-lab-cp01-20261007-030000.db
etcd-snapshot-faekcorp-lab-cp01-20261007-090000.db
```

**Why this format:**
- `YYYYMMDD-HHMMSS` sorts chronologically as text
- No characters that break filenames
- Human-readable
- Machine-parseable

### 12.4 — S3 key structure

Organize S3 keys to match:

```
s3://ha-cluster-s3-etcd-lab/
  └── etcd-snapshots/
      └── 2026/
          └── 10/
              └── 06/
                  └── etcd-snapshot-cp01-20261006-211105.db
```

Or keep it flat:

```
s3://ha-cluster-s3-etcd-lab/etcd-snapshots/
  ├── etcd-snapshot-cp01-20261006-000000.db
  ├── etcd-snapshot-cp01-20261006-060000.db
  ├── etcd-snapshot-cp01-20261006-120000.db
  └── etcd-snapshot-cp01-20261006-180000.db
```

Hierarchical is easier for retention policies (delete the whole `2025/` folder).

### 12.5 — S3 lifecycle policy

S3 can automatically delete or transition objects based on age:

```
Rule: etcd-snapshots
  Transition to S3 Standard-IA      after 7 days
  Transition to Glacier Instant     after 30 days
  Delete                            after 365 days
```

**Retention best practice:**

| Age | Retention | Why |
|-----|-----------|-----|
| 0–24h | Hourly snapshots | Immediate recovery |
| 1–30d | Daily snapshots | Recent disaster recovery |
| 1–12mo | Weekly/monthly | Long-term audit, compliance |

Configure with an S3 lifecycle rule. This is what keeps S3 backups cheap.

---

## 13. Production Considerations

### 13.1 — Automate, but not with Kubernetes CronJob

**Don't** run backups as a Kubernetes CronJob.

**Why:** if the cluster is broken, the CronJob can't run. The backup mechanism must not depend on the system it's backing up.

**Use instead:**
- systemd timer on a CP node
- External scheduler (Jenkins, GitHub Actions, dedicated backup server)
- Cloud-native tools (Velero, Kasten)

### 13.2 — Full automation flow

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

### 13.3 — Least-privilege IAM policy

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

### 13.4 — IAM role vs static keys

Always prefer IAM roles on EC2. Static keys should only be used when there's no alternative (local development, CI runners without OIDC, etc.).

### 13.5 — Verification cadence

| Verification | Frequency |
|--------------|-----------|
| Snapshot creation | Every backup run (automated) |
| Snapshot integrity (`etcdutl snapshot status`) | Every backup run |
| S3 upload integrity (MD5/ETag) | Every backup run |
| **Full restore test** | Quarterly, on isolated cluster |

**A backup that has never been restored is only an assumption.**

### 13.6 — Encryption

| Layer | Method |
|-------|--------|
| At rest in S3 | `--sse AES256` (S3-managed key) or `--sse aws:kms` (KMS-managed) |
| In transit (cp01 → S3) | HTTPS (default for AWS CLI) |
| On cp01 disk (temporary) | File permission 600 (root only) |

For highly regulated environments, use KMS with key rotation and CloudTrail audit.

### 13.7 — Monitoring and alerting

- Alert if backup hasn't run in >12 hours
- Alert if `etcdutl snapshot status` fails
- Alert if S3 upload fails
- Alert if snapshot size drops suddenly (possible data loss)
- Alert on unexpected S3 access (potential breach)

### 13.8 — What the backup does NOT contain

- Container images
- Application data on PersistentVolumes (databases, file uploads)
- Cloud provider resources (ELBs, EBS volumes, S3 buckets)
- External systems (DNS, certificate managers)
- Certificates and private keys (well — the ones in Kubernetes Secrets are there, but the etcd PKI itself is not)

**You need separate backup strategies for each of those.**

---

## 14. Complete Command Reference

### Quick copy-paste block (for future use)

```bash
# ============================================
# On cp01 — etcd backup workflow
# ============================================

# 1. Verify cluster health
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health -w table

# 2. Take snapshot
sudo etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/snapshot.db

# 3. Verify integrity
sudo etcdutl snapshot status /var/lib/etcd/snapshot.db -w table

# 4. Copy with timestamped name
sudo cp /var/lib/etcd/snapshot.db \
  /home/ubuntu/etcd-snapshot-cp01-$(date +%Y%m%d-%H%M%S).db
sudo chown ubuntu:ubuntu /home/ubuntu/etcd-snapshot-cp01-*.db
sudo chmod 600 /home/ubuntu/etcd-snapshot-cp01-*.db

# 5. Verify the copy
sudo etcdutl snapshot status /home/ubuntu/etcd-snapshot-cp01-*.db -w table

# 6. Upload to S3 with encryption
aws s3 cp /home/ubuntu/etcd-snapshot-cp01-*.db \
  s3://ha-cluster-s3-etcd-lab/etcd-snapshots/ \
  --sse AES256

# 7. Verify upload
aws s3 ls s3://ha-cluster-s3-etcd-lab/etcd-snapshots/

# 8. (Optional) Cryptographic verification
md5sum /home/ubuntu/etcd-snapshot-cp01-*.db
aws s3api head-object \
  --bucket ha-cluster-s3-etcd-lab \
  --key etcd-snapshots/$(basename /home/ubuntu/etcd-snapshot-cp01-*.db) \
  --query ETag --output text

# 9. Cleanup local
sudo rm /var/lib/etcd/snapshot.db
rm /home/ubuntu/etcd-snapshot-cp01-*.db
```

### Reference table — all commands

| Command | Purpose |
|---------|---------|
| `etcdctl version` | Verify etcdctl installed |
| `etcdutl version` | Verify etcdutl installed |
| `etcdctl member list` | List etcd members |
| `etcdctl endpoint health` | Check etcd health |
| `etcdctl snapshot save <path>` | Create backup snapshot |
| `etcdutl snapshot status <path>` | Verify snapshot integrity |
| `etcdutl snapshot restore <path>` | Restore (separate runbook) |
| `aws sts get-caller-identity` | Verify AWS identity |
| `aws s3 ls <bucket>/` | List S3 objects |
| `aws s3 cp <local> <s3>` | Upload to S3 |
| `aws s3api head-object` | Get S3 object metadata (ETag) |
| `sha256sum <file>` | Local file hash |
| `md5sum <file>` | Local file MD5 |

### Reference table — key paths

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

---

## Summary — The Whole Flow at a Glance

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
                                 │  upload
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
