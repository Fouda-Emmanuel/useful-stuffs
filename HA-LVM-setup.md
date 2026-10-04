# Kubernetes Node Storage Setup — LVM + `/data` for containerd & kubelet


**Node:** `cp01.faekcorp.lab` (`10.70.21.6`)
**Extra disk:** `/dev/nvme1n1` (60 GB)
**Goal:** Move container runtime and kubelet state off the OS disk onto a dedicated LVM-backed filesystem.

---

## 📖 Before You Start — Read This

**The problem we're solving:**

By default, Kubernetes stores two big directories on the OS disk:

- `/var/lib/containerd` → container images, layers, snapshots
- `/var/lib/kubelet` → pod state, volume mounts, plugin data

If these fill up the OS disk, the **node dies**. So we move them to a dedicated disk.

**What you'll build:**

```
Extra disk → LVM → ext4 → /data → symlinks back to /var/lib/...
```

**Time required:** ~15 minutes per node.

**Prerequisites:**

- SSH access with `sudo`
- A second empty disk attached (here: `/dev/nvme1n1`, 60 GB)
- The OS disk stays untouched (here: `/dev/nvme0n1`, 20 GB)

---

## 0. Introduction — What are we doing, and why?

### 0.1 The problem in plain English

Every Kubernetes node runs two big pieces of software that eat disk space:

| Component | Default directory | What it stores | Grows how big? |
|-----------|-------------------|----------------|----------------|
| **containerd** | `/var/lib/containerd` | Container images, image layers, snapshots, container metadata | Can reach tens of GB |
| **kubelet** | `/var/lib/kubelet` | Pod state, volume mounts, secrets/configmaps on disk, plugin state | Several GB, grows with pods |

By default, both live on the **operating system disk** (`/dev/nvme0n1`, only 20 GB on this node).

That's a problem. Imagine:

```
OS disk (20 GB)
├── Ubuntu system        ~5 GB
├── logs, apt cache      ~2 GB
├── /var/lib/containerd  ~10 GB  ← grows as you pull images
└── /var/lib/kubelet     ~3 GB   ← grows as you run pods
                       ─────────
                       = ~20 GB → DISK FULL
```

When the OS disk fills up:

- `kubelet` crashes
- New pods can't start
- Logs can't be written
- The whole node becomes unstable
- SSH might even fail

This is a **classic Kubernetes outage cause** in production.

### 0.2 The solution

Attach a **separate disk** (60 GB here), and move `containerd` + `kubelet` storage onto it. The OS disk stays small and clean; Kubernetes gets its own dedicated storage.

```
OS disk (20 GB)              Extra disk (60 GB)
├── Ubuntu                   └── /data
├── logs                         ├── /data/containerd
├── apt cache                    └── /data/kubelet
└── nothing Kubernetes
```

If `/data` fills up, Kubernetes is affected — but the **OS stays alive**, so you can still SSH in and fix it. That's a much better failure mode.

### 0.3 Why not just format the whole disk with ext4?

You could do:

```
/dev/nvme1n1 → ext4 → /data
```

But then:

- You can't easily shrink or extend it
- You can't split it into multiple volumes
- You can't add a second disk later to expand
- You can't take snapshots

**LVM** solves all of this.

### 0.4 What is LVM?

**LVM** = Logical Volume Manager. It's a Linux storage layer that sits **between** the physical disk and the filesystem.

Instead of:

```
Disk → Filesystem
```

You get:

```
Disk → LVM → Filesystem
```

LVM adds three layers:

| Layer | Full name | What it is | Our value |
|-------|-----------|------------|-----------|
| **PV** | Physical Volume | A disk marked as usable by LVM | `/dev/nvme1n1` |
| **VG** | Volume Group | A pool of one or more PVs | `vgk8s` (60 GB) |
| **LV** | Logical Volume | A slice of the VG, presented as a block device | `lvroot` (50 GB) |

**Analogy — the pizza:**

- The **PV** is the whole 60 GB pizza.
- The **VG** is the table that holds the pizza (could hold more pizzas later).
- The **LV** is one slice you cut from it (50 GB).
- The **filesystem** (ext4) is what makes the slice edible — usable for files.
- The **mount point** (`/data`) is where you place it on the filesystem tree.

**Why this design helps:**

- Need more space later? Extend the LV (`lvextend`) instead of rebuilding.
- Need a second volume (e.g., for etcd)? Carve another LV from the same VG.
- Need to add a new disk to the pool? `vgextend vgk8s /dev/new-disk`.

That's why we use LVM instead of plain partitions.

### 0.5 The tools we'll use

| Tool | Purpose | Where it comes from |
|------|---------|---------------------|
| `lsblk` | List block devices (disks, partitions, LVs, mounts) | `util-linux` (preinstalled) |
| `fdisk -l` | Show disk partition details | `util-linux` (preinstalled) |
| `pvcreate` | Create LVM Physical Volume | `lvm2` package |
| `vgcreate` | Create LVM Volume Group | `lvm2` package |
| `lvcreate` | Create LVM Logical Volume | `lvm2` package |
| `pvs` / `vgs` / `lvs` | Show LVM PVs / VGs / LVs | `lvm2` package |
| `lvdisplay` | Detailed LV info | `lvm2` package |
| `mkfs.ext4` | Create an ext4 filesystem | `e2fsprogs` (preinstalled) |
| `blkid` | Show UUID / filesystem type of a device | `util-linux` (preinstalled) |
| `mount` | Attach a filesystem to a directory | `util-linux` (preinstalled) |
| `findmnt` | Show where a filesystem is mounted | `util-linux` (preinstalled) |
| `df` | Show disk usage | `coreutils` (preinstalled) |
| `ln -s` | Create a symbolic link | `coreutils` (preinstalled) |

> 💡 **Is `lvm2` installed?** On Ubuntu, the `lvm2` package is usually preinstalled on server images. Verify with:
> ```bash
> which pvcreate vgcreate lvcreate
> ```
> If missing, install:
> ```bash
> sudo apt update && sudo apt install -y lvm2
> ```

### 0.6 The layers we will build — one picture

```
    Physical disk
   /dev/nvme1n1 (60 GB)
            │
            │  pvcreate     →  "LVM can manage this disk"
            ▼
    LVM Physical Volume (PV)
            │
            │  vgcreate     →  "Pool the capacity as vgk8s"
            ▼
    Volume Group vgk8s (60 GB)
            │
            │  lvcreate     →  "Carve out 50 GB as lvroot"
            ▼
    Logical Volume lvroot (50 GB)
            │
            │  mkfs.ext4    →  "Put a filesystem on the LV"
            ▼
    ext4 filesystem
            │
            │  mount        →  "Attach it at /data"
            ▼
    /data
            │
            ├── /data/containerd
            └── /data/kubelet
                    │
                    │  ln -s        →  "Redirect the OS paths"
                    ▼
    /var/lib/containerd → /data/containerd
    /var/lib/kubelet    → /data/kubelet
```

Each step builds on the previous one. **Skip nothing** — every layer is verified before we move on.

### 0.7 What we will NOT do

- ❌ We will **not** format `/dev/nvme1n1` directly.
- ❌ We will **not** touch `/dev/nvme0n1` (the OS disk).
- ❌ We will **not** delete any existing Kubernetes data (unless we check first).

Every step **verifies** what it did before moving on. This is the "measure twice, cut once" approach.

### 0.8 How to read the rest of this document

Each step follows the same structure:

1. **The command** — what to run
2. **Expected output** — what you should see
3. **Verification command** — how to confirm it worked
4. **What just happened** — the concept, so you understand *why*

If something doesn't match, **stop and troubleshoot** before continuing. Never push through an unexpected result on storage commands.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Why LVM and not just a partition?](#2-why-lvm-and-not-just-a-partition)
3. [Step 0 — Safety check](#step-0--safety-check)
4. [Step 1 — Create the LVM Physical Volume](#step-1--create-the-lvm-physical-volume)
5. [Step 2 — Create the Volume Group](#step-2--create-the-volume-group)
6. [Step 3 — Create the Logical Volume](#step-3--create-the-logical-volume)
7. [Step 4 — Verify the new device](#step-4--verify-the-new-device)
8. [Step 5 — Create the ext4 filesystem](#step-5--create-the-ext4-filesystem)
9. [Step 6 — Create `/data`](#step-6--create-data)
10. [Step 7 — Mount the LV at `/data`](#step-7--mount-the-lv-at-data)
11. [Step 8 — Make `/data` persistent via `/etc/fstab`](#step-8--make-data-persistent-via-etcfstab)
12. [Step 9 — Test fstab without rebooting](#step-9--test-fstab-without-rebooting)
13. [Step 10 — Create containerd & kubelet directories](#step-10--create-containerd--kubelet-directories)
14. [Step 11 — Check existing `/var/lib` directories](#step-11--check-existing-varlib-directories)
15. [Step 12 — Create the symlinks](#step-12--create-the-symlinks)
16. [Step 13 — Final verification](#step-13--final-verification)
17. [Step 14 — Reboot test (recommended)](#step-14--reboot-test-recommended)
18. [Step 15 — Extend `lvroot` using free VG space](#step-15--extend-lvroot-using-free-vg-space)
19. [Step 16 — Add a new disk to the VG and extend](#step-16--add-a-new-disk-to-the-vg-and-extend)
20. [Step 17 — Rollback / undo everything](#step-17--rollback--undo-everything)
21. [Step 18 — Monitoring and maintenance](#step-18--monitoring-and-maintenance)
22. [Key Concepts Explained](#key-concepts-explained)
23. [Glossary](#glossary)

---

## 1. Architecture Overview

This is what we're building — **layers**, each one on top of the last:

```
Physical disk
/dev/nvme1n1 (60 GB)
        │
        │ pvcreate
        ▼
LVM Physical Volume (PV)
        │
        │ vgcreate
        ▼
Volume Group (VG) — vgk8s (60 GB)
        │
        │ lvcreate -L 50G
        ▼
Logical Volume (LV) — lvroot (50 GB)
        │
        │ mkfs.ext4
        ▼
ext4 filesystem
        │
        │ mount
        ▼
/data
        │
        ├── /data/containerd
        └── /data/kubelet
                │
                │ symlinks
                ▼
        /var/lib/containerd → /data/containerd
        /var/lib/kubelet    → /data/kubelet
```

### Disk layout on cp01

```
/dev/nvme0n1 (20 GB) — OS disk
├─nvme0n1p1  19 GB → /
├─nvme0n1p14  4 MB
├─nvme0n1p15 106 MB → /boot/efi
└─nvme0n1p16 913 MB → /boot

/dev/nvme1n1 (60 GB) — EMPTY, our Kubernetes storage disk
```

**Never touch `nvme0n1`.** That's the OS.

---

## 2. Why LVM and not just a partition?

The course could have done:

```
/dev/nvme1n1 → /dev/nvme1n1p1 → ext4 → /data
```

But it uses LVM instead. **Why?**

| | Plain partition | LVM |
|---|---|---|
| Resize later | Painful | Easy |
| Add more disks to pool | No | Yes (`vgextend`) |
| Split into multiple volumes | No | Yes |
| Snapshots | No | Yes |

LVM puts a **flexible storage-management layer** between the physical disk and the filesystem. On this node we only use 50 GB of the 60 GB disk — the remaining ~10 GB stays available inside the VG for future use (e.g., extending `lvroot`, or creating a separate LV for etcd).

---

## Step 0 — Safety check

**Before touching anything**, confirm which disk is which.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo fdisk -l /dev/nvme1n1
sudo pvs
```

Expected:

```
nvme1n1   60G  disk          ← no partitions, no filesystem, no mount
nvme0n1   20G  disk          ← has partitions, mounted at /
```

> 🧠 **Why this matters:** Every command from here on modifies `/dev/nvme1n1`. If you accidentally target `/dev/nvme0n1`, you destroy the OS. Always double-check the device name.

> 💡 **On AWS:** NVMe device names depend on the instance type and volume attachment order. `/dev/nvme0n1` is usually the root volume; `/dev/nvme1n1` is usually the first additional volume.

---

## Step 1 — Create the LVM Physical Volume

```bash
sudo pvcreate /dev/nvme1n1
```

Expected:

```
Physical volume "/dev/nvme1n1" successfully created.
```

Verify:

```bash
sudo pvs
```

Expected:

```
PV             VG   Fmt  Attr PSize   PFree
/dev/nvme1n1        lvm2 ---  <60.00g <60.00g
```

**What just happened:**

`pvcreate` wrote **LVM metadata** at the start of the disk. The disk is unchanged in size and name — it's now just marked as "LVM can manage me."

| Before | After |
|--------|-------|
| Raw 60 GB block device | LVM Physical Volume |
| Not usable as a filesystem | Still not usable — no filesystem yet |
| Not in any pool | Not in any VG yet |

> 🧠 **Key point:** `pvcreate` does **not** format the disk. It does **not** create a filesystem. It just prepares it for LVM.

---

## Step 2 — Create the Volume Group

```bash
sudo vgcreate vgk8s /dev/nvme1n1
```

Expected:

```
Volume group "vgk8s" successfully created
```

Verify:

```bash
sudo vgs
sudo pvs
```

Expected:

```
VG     #PV #LV #SN Attr   VSize   VFree
vgk8s    1   0   0 wz--n- <60.00g <60.00g

PV             VG    Fmt  Attr PSize   PFree
/dev/nvme1n1   vgk8s lvm2 a--  <60.00g <60.00g
```

**What just happened:**

The VG (`vgk8s`) is a **storage pool**. Right now it holds all 60 GB, unallocated.

```
/dev/nvme1n1
      │
      ▼
vgk8s (60 GB pool)
      │
      └── (nothing allocated yet)
```

> 🧠 **Why `vgk8s`?** It's a naming convention meaning "volume group for Kubernetes." You could call it anything — but pick a meaningful name for the environment.

---

## Step 3 — Create the Logical Volume

```bash
sudo lvcreate -n lvroot -L 50G vgk8s
```

Expected:

```
Logical volume "lvroot" created.
```

**Break down the command:**

| Piece | Meaning |
|-------|---------|
| `lvcreate` | Create a logical volume |
| `-n lvroot` | Name it `lvroot` |
| `-L 50G` | Size: 50 GB |
| `vgk8s` | Take space from this VG |

Verify:

```bash
sudo lvs
sudo vgs
```

Expected:

```
LV     VG    Attr       LSize
lvroot vgk8s -wi-a----- 50.00g

VG     #PV #LV #SN Attr   VSize   VFree
vgk8s    1   1   0 wz--n- <60.00g <10.00g
```

**What just happened:**

We carved 50 GB out of the 60 GB pool into a logical block device. About **10 GB remains free** inside the VG.

```
vgk8s (60 GB)
├── lvroot     50 GB  ← allocated
└── FREE       10 GB  ← reserved for future use
```

> 💡 **Why only 50 GB?** Leaving free space in the VG gives you flexibility later — you can extend `lvroot`, or create another LV (e.g., for etcd data), without adding a new disk.

> ⚠️ **Name trap:** The LV is called `lvroot`, but it is **not** the Linux root filesystem. Your root filesystem is `/dev/nvme0n1p1`. `lvroot` is just a name — it's mounted at `/data`, not `/`. It could have been called `lvdata` or `lvk8s` with no functional difference.

---

## Step 4 — Verify the new device

```bash
lsblk
sudo lvdisplay
```

Expected (from `lsblk`):

```
nvme1n1        60G disk
└─vgk8s-lvroot 50G lvm
```

The LV appears as a child of the disk. Its device paths:

- `/dev/vgk8s/lvroot`
- `/dev/mapper/vgk8s-lvroot`

**Both point to the same logical volume.**

> 🧠 **How can `mkfs` work on an LV?** Because Linux presents an LV as a normal **block device**. `mkfs` doesn't care whether a block device is physical or logical — it just writes a filesystem.

---

## Step 5 — Create the ext4 filesystem

**Now** we format — but the **LV**, not the disk.

```bash
sudo mkfs.ext4 /dev/vgk8s/lvroot
```

> ⚠️ **Do NOT run `mkfs.ext4 /dev/nvme1n1`.** That would put a filesystem directly on the whole disk, bypassing LVM entirely — a totally different design.

Expected:

```
Creating filesystem with 13107200 4k blocks and 3276800 inodes
Filesystem UUID: a76026d5-4604-4051-9bee-a665355742eb
...
done
```

**Note the UUID** — we'll need it for `/etc/fstab`.

Verify:

```bash
lsblk -f
sudo blkid /dev/vgk8s/lvroot
```

Expected:

```
nvme1n1
└─vgk8s-lvroot  ext4  1.0  a76026d5-...

/dev/vgk8s/lvroot: UUID="a76026d5-4604-4051-9bee-a665355742eb" TYPE="ext4"
```

> 🧠 **What is ext4?** A **filesystem** — the structure that tells Linux how to organize files, directories, permissions, and free space on a block device. Without a filesystem, the LV is just raw capacity; Linux can't store files on it.

**LV vs filesystem:**

| Layer | What it is |
|-------|-----------|
| LV (`lvroot`) | Block device — raw capacity |
| ext4 | Filesystem structure — usable for files |

---

## Step 6 — Create `/data`

```bash
sudo mkdir -p /data
```

Verify:

```bash
ls -ld /data
```

Expected:

```
drwxr-xr-x 2 root root 4096 ... /data
```

Right now `/data` is **just an ordinary directory on the OS disk** (root filesystem). Nothing special yet.

---

## Step 7 — Mount the LV at `/data`

```bash
sudo mount /dev/vgk8s/lvroot /data
```

Verify:

```bash
findmnt /data
df -h /data
```

Expected:

```
TARGET  SOURCE                   FSTYPE OPTIONS
/data   /dev/mapper/vgk8s-lvroot ext4   rw,relatime

Filesystem              Size  Used Avail Use% Mounted on
/dev/mapper/vgk8s-lvroot 49G   24K   47G   1% /data
```

**What just happened:**

The ext4 filesystem on `lvroot` is now visible at `/data`. Anything you write under `/data` goes to the **60 GB disk**, not the OS disk.

> 🧠 **Shadowing effect:** When you mount a filesystem over an existing directory, the original contents of that directory are **hidden underneath** (not deleted). If `/data` had a file before mounting, that file is inaccessible until you unmount — but it's still there on disk.

---

## Step 8 — Make `/data` persistent via `/etc/fstab`

`mount` on its own is **runtime only**. Reboot the machine and `/data` won't be mounted. We need `/etc/fstab`.

### 8.1 Get the UUID

```bash
UUID=$(sudo blkid -s UUID -o value /dev/vgk8s/lvroot)
echo "$UUID"
```

Expected:

```
a76026d5-4604-4051-9bee-a665355742eb
```

**Break down `blkid -s UUID -o value`:**

- `-s UUID` → show only the UUID field
- `-o value` → output only the value, no `UUID=` prefix

> 🧠 **Why UUID instead of `/dev/vgk8s/lvroot`?** Because device paths can change (renumbering, adding disks), but a UUID identifies the **filesystem itself**. `/etc/fstab` is far safer using UUID.

### 8.2 Inspect the current fstab

```bash
cat /etc/fstab
```

You'll likely see entries for the root, boot, and EFI partitions. Don't touch those.

### 8.3 Append the new entry

```bash
echo "UUID=$UUID /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

Verify:

```bash
tail -n 5 /etc/fstab
```

Expected:

```
UUID=a76026d5-4604-4051-9bee-a665355742eb /data ext4 defaults 0 2
```

**Field breakdown:**

| Field | Value | Meaning |
|-------|-------|---------|
| Device | `UUID=...` | Which filesystem |
| Mount point | `/data` | Where to mount |
| Type | `ext4` | Filesystem type |
| Options | `defaults` | Standard mount options (`rw`, `suid`, `dev`, `exec`, `auto`, `nouser`, `async`) |
| Dump | `0` | Legacy backup tool — ignore |
| FSCK order | `2` | Check after root (`1`); `0` means never check |

---

## Step 9 — Test fstab without rebooting

> 🧠 **Golden rule:** Never reboot to test an fstab change. If it's broken, the machine may not boot. Use `mount -a` first.

```bash
sudo mount -a
```

If this produces **no output**, the fstab entry is valid. If it prints errors, fix `/etc/fstab` before rebooting.

Verify:

```bash
findmnt /data
df -h /data
```

Same output as before. The mount works both ways.

---

## Step 10 — Create containerd & kubelet directories

```bash
sudo mkdir -p /data/containerd
sudo mkdir -p /data/kubelet
```

Verify:

```bash
ls -la /data
```

Expected:

```
containerd
kubelet
lost+found
```

**What goes in each directory:**

| Directory | Used by | Contains |
|-----------|---------|----------|
| `/data/containerd` | containerd | Images, layers, snapshots, container metadata |
| `/data/kubelet` | kubelet | Pod state, volume mounts, plugin state |

> 🧠 `lost+found` is created automatically by `mkfs.ext4`. It's normal — leave it alone.

---

## Step 11 — Check existing `/var/lib` directories

**Before** deleting anything, check what's there:

```bash
sudo ls -ld /var/lib/containerd /var/lib/kubelet
sudo du -sh /var/lib/containerd /var/lib/kubelet 2>/dev/null
```

On our node, both were **absent** (fresh OS, containerd/kubelet not yet installed) — so `ls` returned "No such file or directory."

> ⚠️ **If those directories exist and contain data, STOP.** On a live Kubernetes node, deleting them destroys container images and pod state. On a fresh node (this case), it's harmless.

---

## Step 12 — Create the symlinks

We want Kubernetes to keep using its normal paths (`/var/lib/containerd`, `/var/lib/kubelet`), but the actual data to live under `/data`. The cleanest way is **symlinks**.

```bash
sudo rm -rf /var/lib/containerd
sudo rm -rf /var/lib/kubelet

sudo ln -s /data/containerd /var/lib/containerd
sudo ln -s /data/kubelet    /var/lib/kubelet
```

Verify:

```bash
ls -ld /var/lib/containerd /var/lib/kubelet
```

Expected:

```
/var/lib/containerd -> /data/containerd
/var/lib/kubelet    -> /data/kubelet
```

**What this means:**

When containerd writes to `/var/lib/containerd/...`, the kernel follows the symlink:

```
/var/lib/containerd
       ↓ symlink
/data/containerd
       ↓
/data (ext4 on lvroot)
       ↓
lvroot LV (50 GB)
       ↓
vgk8s VG (60 GB)
       ↓
/dev/nvme1n1 (60 GB physical disk)
```

Same for kubelet.

> 🧠 **Why symlinks and not config changes?** Because containerd and kubelet are configured by default to use `/var/lib/...`. Changing that requires editing multiple config files. Symlinks are the transparent, zero-config solution.

> ⚠️ **`rm -rf` warning:** The `-r` means recursive, `-f` means force. **Never** run this against a path with data you need. On our fresh node it was safe because those paths didn't exist.

---

## Step 13 — Final verification

Run this block to check every layer:

```bash
echo "=== BLOCK DEVICES ==="
lsblk -f

echo
echo "=== LVM PV ==="
sudo pvs

echo
echo "=== LVM VG ==="
sudo vgs

echo
echo "=== LVM LV ==="
sudo lvs

echo
echo "=== MOUNT ==="
findmnt /data

echo
echo "=== DISK USAGE ==="
df -h /data

echo
echo "=== SYMLINKS ==="
ls -ld /var/lib/containerd /var/lib/kubelet

echo
echo "=== FSTAB ==="
grep -E '[[:space:]]/data[[:space:]]' /etc/fstab
```

Expected summary:

```
nvme1n1        60G disk
└─vgk8s-lvroot 50G lvm /data

PV           VG    PSize   PFree
/dev/nvme1n1 vgk8s <60G    <10G

VG    VSize   VFree
vgk8s <60G    <10G

LV     LSize
lvroot 50G

TARGET SOURCE                   FSTYPE
/data  /dev/mapper/vgk8s-lvroot ext4

/dev/mapper/vgk8s-lvroot 49G ... /data

/var/lib/containerd -> /data/containerd
/var/lib/kubelet    -> /data/kubelet

UUID=a76026d5-... /data ext4 defaults 0 2
```

If all that matches, **the node is ready for Kubernetes.**

---

## Step 14 — Reboot test (recommended)

> 🧠 **Why reboot?** Because `mount -a` proves fstab syntax, but a reboot proves the **full boot-time flow** works. Do this **before** installing containerd/kubelet — so if something's wrong, you're not debugging Kubernetes at the same time.

```bash
sudo reboot
```

Reconnect, then check:

```bash
lsblk
findmnt /data
df -h /data
ls -ld /var/lib/containerd /var/lib/kubelet
sudo pvs
sudo vgs
sudo lvs
```

You want to see:

- `nvme1n1` → `vgk8s-lvroot 50G lvm /data`
- `/data` mounted from `/dev/mapper/vgk8s-lvroot`
- Symlinks still present
- LVM layers still present (`vgk8s`, `lvroot`, `~10 GB free`)

If everything survived — the storage foundation is **locked in**.

---

## Step 15 — Extend `lvroot` using free VG space

### 15.1 Scenario

You still have **~10 GB free** inside `vgk8s`. Let's say containerd is filling up `/data` and you want to grow it.

```bash
# 1. Check how much free space is in the VG
sudo vgs
```

Expected:

```
VG    VSize   VFree
vgk8s <60G    <10G
```

### 15.2 Extend the LV

Two ways — pick one:

**Option A — Extend by a fixed amount (e.g., +5 GB):**

```bash
sudo lvextend -L +5G /dev/vgk8s/lvroot
```

**Option B — Use ALL remaining free space:**

```bash
sudo lvextend -l +100%FREE /dev/vgk8s/lvroot
```

> 🧠 **`-L +5G`** = "add 5 GB more"
> **`-l +100%FREE`** = "use all free extents in the VG" (lowercase `-l` for extents, uppercase `-L` for size)

### 15.3 Grow the filesystem

Extending the LV makes the **block device** bigger, but the **filesystem** on it doesn't automatically grow. You must tell ext4 to expand:

```bash
sudo resize2fs /dev/vgk8s/lvroot
```

> 💡 **Why `resize2fs`?** The filesystem metadata (block count, inode tables, etc.) was sized for the old LV. `resize2fs` rewrites it to match the new LV size. ext4 supports **online resize** — no unmount needed.

### 15.4 Verify

```bash
df -h /data
sudo lvs
sudo vgs
```

You should see `/data` now reports the larger size, and `VFree` in the VG shrank accordingly.

### 15.5 Full example — extending by all free space

```bash
# Before
df -h /data             # ~49G
sudo vgs                # VFree ~10G

# Extend LV to use all free VG space
sudo lvextend -l +100%FREE /dev/vgk8s/lvroot

# Grow the filesystem
sudo resize2fs /dev/vgk8s/lvroot

# After
df -h /data             # ~59G
sudo vgs                # VFree 0
```

> ⚠️ **Note:** `-l +100%FREE` consumes all remaining space in the VG. If you might need space for another LV later (e.g., etcd), use `-L +5G` instead.

---

## Step 16 — Add a new disk to the VG and extend

### 16.1 Scenario

You've used up all 60 GB, `/data` is full, and you've attached a **new disk** (e.g., `/dev/nvme2n1`, 40 GB) to the instance.

> 💡 **On AWS:** Attach the new EBS volume in the console, then it appears as `/dev/nvme2n1` (or similar). Confirm with `lsblk`.

### 16.2 Verify the new disk is visible

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Expected:

```
nvme2n1   40G  disk   ← new, no filesystem, no partitions
```

### 16.3 Prepare the new disk as a PV

```bash
sudo pvcreate /dev/nvme2n1
```

Verify:

```bash
sudo pvs
```

Expected:

```
PV             VG    PSize   PFree
/dev/nvme1n1   vgk8s <60G    ...
/dev/nvme2n1         <40G    <40G   ← new, not yet in a VG
```

### 16.4 Add the new PV to the existing VG

```bash
sudo vgextend vgk8s /dev/nvme2n1
```

Verify:

```bash
sudo vgs
```

Expected:

```
VG    #PV #LV #SN Attr   VSize   VFree
vgk8s   2   1   0 wz--n- <100G   <40G
```

> 🧠 **What just happened:** The VG now spans **two physical disks**. From LVM's perspective, it doesn't matter which physical disk the extents live on — it just sees a bigger pool.

### 16.5 Extend the LV across the new space

```bash
sudo lvextend -l +100%FREE /dev/vgk8s/lvroot
sudo resize2fs /dev/vgk8s/lvroot
```

Verify:

```bash
df -h /data
sudo lvs
sudo vgs
```

`/data` is now ~99 GB. **The mount was never interrupted** — this all happened online.

### 16.6 Diagram — before and after

**Before:**
```
vgk8s (60 GB)
└── lvroot 50 GB
   /dev/nvme1n1
```

**After:**
```
vgk8s (100 GB, spans 2 disks)
└── lvroot ~99 GB
   ├── /dev/nvme1n1 (60 GB)
   └── /dev/nvme2n1 (40 GB)
```

### 16.7 Migrating data off an old disk (optional)

If you ever need to **remove** a disk from the VG (e.g., to swap a failing disk), you can move extents off it first:

```bash
sudo pvmove /dev/nvme1n1
```

This moves all LVM extents from `nvme1n1` to other PVs in the same VG. **Live, no downtime** (for most workloads). After it completes:

```bash
sudo vgreduce vgk8s /dev/nvme1n1
sudo pvremove /dev/nvme1n1
```

> ⚠️ **Warning:** `pvmove` can take a long time on large volumes. Don't interrupt it.

---

## Step 17 — Rollback / undo everything

> ⚠️ **Serious warning:** These commands **destroy data**. Only run them on a node you're intentionally decommissioning, or when you're sure the data on `/data` is disposable (e.g., a fresh node that never had Kubernetes on it).

### 17.1 Safety checklist before rollback

Answer these **before** proceeding:

1. Is Kubernetes running on this node?
   - ✅ If **yes** → **stop `kubelet` and `containerd` first**, or better, `kubeadm reset` and drain the node.
   - ✅ If **no** → safe to proceed.
2. Is `/data` holding anything important?
   - ✅ If **yes** → copy it elsewhere first.
3. Is the OS disk on a different device? (Confirm with `lsblk`.)
   - ✅ Should be — otherwise you're about to erase the OS.

**Stop Kubernetes services (if running):**

```bash
sudo systemctl stop kubelet
sudo systemctl stop containerd
sudo kubeadm reset -f 2>/dev/null || true
```

### 17.2 Remove the symlinks

```bash
sudo rm -f /var/lib/containerd
sudo rm -f /var/lib/kubelet
```

> ⚠️ Use `-f`, **not** `-rf`, when removing symlinks. `rm -rf` on a symlink directory could traverse into `/data` (depending on shell behavior) — safer to remove just the link.

Recreate empty directories for a clean state:

```bash
sudo mkdir -p /var/lib/containerd
sudo mkdir -p /var/lib/kubelet
```

### 17.3 Remove the `/etc/fstab` entry

```bash
sudo cp /etc/fstab /etc/fstab.bak
sudo sed -i '\|/data|d' /etc/fstab
```

Verify:

```bash
grep /data /etc/fstab     # should print nothing
cat /etc/fstab            # ensure other lines are intact
```

> 🧠 **Why `sed` with `|` delimiter?** Because the pattern contains slashes (`/data`). Using `|` avoids escaping them.

### 17.4 Unmount `/data`

```bash
sudo umount /data
```

Verify:

```bash
findmnt /data             # should print nothing
```

If `umount` fails with "target is busy":

```bash
sudo lsof +D /data        # see what's using it
sudo fuser -m /data       # alternative
```

Stop the offending processes, then retry.

### 17.5 Remove the LV

```bash
sudo lvremove /dev/vgk8s/lvroot
```

You'll be prompted:

```
Do you really want to remove active logical volume vgk8s/lvroot? [y/n]: y
```

For scripting, add `-f`:

```bash
sudo lvremove -f /dev/vgk8s/lvroot
```

Verify:

```bash
sudo lvs                   # lvroot should be gone
sudo vgs                   # VFree should be back to ~60G
```

### 17.6 Remove the VG

Only do this if you want to **fully dismantle** the pool:

```bash
sudo vgremove vgk8s
```

Verify:

```bash
sudo vgs                   # vgk8s should be gone
```

### 17.7 Remove the PV

```bash
sudo pvremove /dev/nvme1n1
```

Verify:

```bash
sudo pvs                   # nvme1n1 should be gone
```

**Now the disk is back to raw state.** You can re-use it, re-partition it, or detach it in AWS.

### 17.8 Full rollback — one-shot script

> ⚠️ For **fresh nodes only**. Read every line before running.

```bash
# STOP Kubernetes first if it's running
sudo systemctl stop kubelet containerd 2>/dev/null
sudo kubeadm reset -f 2>/dev/null || true

# Remove symlinks
sudo rm -f /var/lib/containerd /var/lib/kubelet
sudo mkdir -p /var/lib/containerd /var/lib/kubelet

# Remove fstab entry
sudo cp /etc/fstab /etc/fstab.bak
sudo sed -i '\|/data|d' /etc/fstab

# Unmount
sudo umount /data 2>/dev/null || true

# Dismantle LVM (bottom-up)
sudo lvremove -f /dev/vgk8s/lvroot
sudo vgremove -f vgk8s
sudo pvremove /dev/nvme1n1

# Remove mount point
sudo rmdir /data 2>/dev/null || true

# Verify
lsblk
sudo pvs ; sudo vgs ; sudo lvs
grep /data /etc/fstab
```

### 17.9 Rollback — quick reference table

| Layer | Creation command | Rollback command |
|-------|------------------|------------------|
| Symlinks | `ln -s /data/containerd /var/lib/containerd` | `rm -f /var/lib/containerd` |
| fstab entry | `echo ... >> /etc/fstab` | `sed -i '\|/data\|d' /etc/fstab` |
| Mount | `mount /dev/vgk8s/lvroot /data` | `umount /data` |
| Filesystem | `mkfs.ext4 /dev/vgk8s/lvroot` | *(destroyed when LV is removed)* |
| Logical Volume | `lvcreate -n lvroot ...` | `lvremove /dev/vgk8s/lvroot` |
| Volume Group | `vgcreate vgk8s ...` | `vgremove vgk8s` |
| Physical Volume | `pvcreate /dev/nvme1n1` | `pvremove /dev/nvme1n1` |

> 🧠 **Order matters:** Rollback is **bottom-up in reverse**: unmount → remove LV → remove VG → remove PV. Creation was **top-down**: PV → VG → LV → FS → mount.

---

## Step 18 — Monitoring and maintenance

### 18.1 Check LVM health at a glance

```bash
sudo pvs    # PVs + their VG
sudo vgs    # VGs + free space
sudo lvs    # LVs + size
lsblk       # whole tree in one picture
```

### 18.2 Check disk usage

```bash
df -h /data
df -h /          # ensure the OS disk isn't filling up
```

### 18.3 Check what's eating `/data`

```bash
sudo du -sh /data/containerd /data/kubelet
sudo du -sh /data/* | sort -h
```

Typical places that grow:

| Path | Cause | Fix |
|------|-------|-----|
| `/data/containerd` | Pulled images | `crictl rmi --prune` |
| `/data/kubelet/pods` | Pod ephemeral storage | Evict misbehaving pods |
| `/data/kubelet/plugins` | CSI/CNI state | Usually stable |

### 18.4 Free space in containerd

```bash
sudo crictl rmi --prune
```

Removes unused images (only if `crictl` is installed and the runtime is up).

### 18.5 Grow when `/data` gets tight

```bash
# Quick check
df -h /data
sudo vgs

# If there's free VG space, extend
sudo lvextend -l +100%FREE /dev/vgk8s/lvroot
sudo resize2fs /dev/vgk8s/lvroot
```

### 18.6 Add capacity — decision tree

```
Is /data full?
│
├── YES ─► Is there free space in vgk8s?
│           │
│           ├── YES ─► lvextend + resize2fs
│           │
│           └── NO  ─► Attach a new disk
│                       │
│                       ├── pvcreate /dev/new-disk
│                       ├── vgextend vgk8s /dev/new-disk
│                       ├── lvextend -l +100%FREE /dev/vgk8s/lvroot
│                       └── resize2fs /dev/vgk8s/lvroot
│
└── NO  ─► Nothing to do
```

### 18.7 Verify LVM is enabled at boot

```bash
systemctl is-enabled lvm2-lvmetad 2>/dev/null || true
systemctl is-enabled lvm2-monitor
```

`lvm2-monitor` should be **enabled** — it activates LVs at boot.

### 18.8 Snapshot before risky changes

Before resizing or moving things around, take an **LVM snapshot** for safety:

```bash
sudo lvcreate -L 5G -s -n lvroot_snap /dev/vgk8s/lvroot
```

> 🧠 **What's a snapshot?** A point-in-time copy. Changes after the snapshot go to a separate area, so you can revert if something goes wrong.

Restore from a snapshot:

```bash
sudo lvconvert --merge /dev/vgk8s/lvroot_snap
```

Remove a snapshot:

```bash
sudo lvremove /dev/vgk8s/lvroot_snap
```

> ⚠️ Snapshots consume VG space. If the VG is full, snapshots fail. Plan for this.

---

## Key Concepts Explained

### Physical Volume (PV) vs Volume Group (VG) vs Logical Volume (LV)

| Layer | Concept | Example here |
|-------|---------|--------------|
| **PV** | A disk LVM can use | `/dev/nvme1n1` |
| **VG** | A pool of PVs | `vgk8s` (60 GB) |
| **LV** | A slice of a VG, presented as a block device | `lvroot` (50 GB) |

**Analogy:** Think of a pizza. The **PV** is the pizza. The **VG** is the whole table (could hold multiple pizzas). The **LV** is a slice you cut.

### Why `mkfs` runs on the LV, not the disk

`/dev/nvme1n1` is the **underlying storage**. `/dev/vgk8s/lvroot` is the **carved-out block device**. Filesystems live on block devices — the LV is one, so it gets formatted. The disk itself is underneath, managed by LVM.

### Why symlinks instead of reconfiguring

containerd and kubelet both default to `/var/lib/<name>`. Symlinks redirect those paths transparently. Zero config changes needed in the applications.

### Why 50 GB out of 60 GB

The remaining **~10 GB stays in the VG** as free space. This lets you:

- Extend `lvroot` later (`lvextend` + `resize2fs`)
- Create an additional LV (e.g., for etcd data)
- Add another disk to the VG without downtime (`vgextend`)

### Why UUID in `/etc/fstab`

Device paths (`/dev/nvme1n1`, `/dev/vgk8s/lvroot`) can theoretically change. UUIDs identify the **filesystem**, and they don't change. Using UUID makes the boot sequence more robust.

### Online resizing

`lvextend` grows the LV while it's mounted. `resize2fs` grows the filesystem while it's mounted. **No downtime, no unmount.**

### Bottom-up creation, top-down rollback

- **Create:** PV → VG → LV → FS → mount
- **Remove:** unmount → FS *(destroyed with LV)* → LV → VG → PV

---

## Glossary

| Term | Meaning |
|------|---------|
| **Block device** | A storage device Linux can read/write in fixed-size blocks |
| **ext4** | The default Linux filesystem |
| **Extent** | The smallest unit of LVM allocation (usually 4 MB) |
| **fstab** | `/etc/fstab` — file listing filesystems to mount at boot |
| **Logical Volume (LV)** | A block device created from a VG |
| **LVM** | Logical Volume Manager — flexible disk management layer |
| **mount** | Attaching a filesystem to a directory |
| **mount point** | The directory a filesystem is mounted on |
| **Physical Volume (PV)** | A disk (or partition) initialized for LVM use |
| **PV move (`pvmove`)** | Move extents from one PV to another in the same VG |
| **Snapshot** | Point-in-time copy of an LV |
| **symlink** | A "shortcut" — a file that points to another path |
| **UUID** | Universally Unique Identifier — a filesystem's permanent ID |
| **Volume Group (VG)** | A pool of one or more PVs, from which LVs are created |

---

## Appendix A — Full command sequence (initial setup)

> Copy-paste friendly. For understanding, read the sections above.

```bash
# ─── 0. Safety check ─────────────────────────────────────
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo fdisk -l /dev/nvme1n1
sudo pvs

# ─── 1. Physical Volume ──────────────────────────────────
sudo pvcreate /dev/nvme1n1
sudo pvs

# ─── 2. Volume Group ─────────────────────────────────────
sudo vgcreate vgk8s /dev/nvme1n1
sudo vgs

# ─── 3. Logical Volume ───────────────────────────────────
sudo lvcreate -n lvroot -L 50G vgk8s
sudo lvs
sudo vgs

# ─── 4. Verify device ────────────────────────────────────
lsblk
sudo lvdisplay

# ─── 5. Filesystem on the LV ─────────────────────────────
sudo mkfs.ext4 /dev/vgk8s/lvroot
lsblk -f
sudo blkid /dev/vgk8s/lvroot

# ─── 6. Create mount point ───────────────────────────────
sudo mkdir -p /data

# ─── 7. Mount ────────────────────────────────────────────
sudo mount /dev/vgk8s/lvroot /data
findmnt /data
df -h /data

# ─── 8. Persist via fstab ────────────────────────────────
UUID=$(sudo blkid -s UUID -o value /dev/vgk8s/lvroot)
echo "$UUID"
cat /etc/fstab
echo "UUID=$UUID /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
tail -n 5 /etc/fstab

# ─── 9. Test fstab ───────────────────────────────────────
sudo mount -a
findmnt /data

# ─── 10. Create Kubernetes directories ──────────────────
sudo mkdir -p /data/containerd
sudo mkdir -p /data/kubelet
ls -la /data

# ─── 11. Check existing paths ────────────────────────────
sudo ls -ld /var/lib/containerd /var/lib/kubelet
sudo du -sh /var/lib/containerd /var/lib/kubelet 2>/dev/null

# ─── 12. Symlinks ────────────────────────────────────────
sudo rm -rf /var/lib/containerd
sudo rm -rf /var/lib/kubelet
sudo ln -s /data/containerd /var/lib/containerd
sudo ln -s /data/kubelet    /var/lib/kubelet
ls -ld /var/lib/containerd /var/lib/kubelet

# ─── 13. Final verification ──────────────────────────────
lsblk -f
sudo pvs && sudo vgs && sudo lvs
findmnt /data
df -h /data
grep -E '[[:space:]]/data[[:space:]]' /etc/fstab

# ─── 14. Reboot test ─────────────────────────────────────
sudo reboot
# After reconnect:
lsblk
findmnt /data
ls -ld /var/lib/containerd /var/lib/kubelet
```

---

## Appendix B — Common operations quick reference

```bash
# ─── Extend LV using free VG space ──────────────────────
sudo vgs
sudo lvextend -l +100%FREE /dev/vgk8s/lvroot
sudo resize2fs /dev/vgk8s/lvroot
df -h /data

# ─── Add a new disk to the VG ───────────────────────────
lsblk                                        # find new disk name
sudo pvcreate /dev/nvme2n1
sudo vgextend vgk8s /dev/nvme2n1
sudo lvextend -l +100%FREE /dev/vgk8s/lvroot
sudo resize2fs /dev/vgk8s/lvroot
df -h /data

# ─── Snapshot before risky change ───────────────────────
sudo lvcreate -L 5G -s -n lvroot_snap /dev/vgk8s/lvroot
sudo lvs

# ─── Restore a snapshot ─────────────────────────────────
sudo umount /data
sudo lvconvert --merge /dev/vgk8s/lvroot_snap
sudo mount /dev/vgk8s/lvroot /data

# ─── Move extents off a disk (to remove it) ─────────────
sudo pvmove /dev/nvme1n1
sudo vgreduce vgk8s /dev/nvme1n1
sudo pvremove /dev/nvme1n1

# ─── Rollback everything (fresh node only!) ─────────────
sudo systemctl stop kubelet containerd 2>/dev/null
sudo rm -f /var/lib/containerd /var/lib/kubelet
sudo cp /etc/fstab /etc/fstab.bak
sudo sed -i '\|/data|d' /etc/fstab
sudo umount /data 2>/dev/null
sudo lvremove -f /dev/vgk8s/lvroot
sudo vgremove -f vgk8s
sudo pvremove /dev/nvme1n1
sudo rmdir /data 2>/dev/null
lsblk ; sudo pvs ; sudo vgs ; sudo lvs
```

---

## Appendix C — Troubleshooting

### Problem: `umount /data` fails with "target is busy"

```bash
sudo lsof +D /data
sudo fuser -m /data
```

Stop the offending processes (usually containerd/kubelet). If they're running:

```bash
sudo systemctl stop kubelet
sudo systemctl stop containerd
sudo umount /data
```

### Problem: `pvcreate` fails with "Device is mounted"

Confirm the disk isn't in use:

```bash
findmnt | grep nvme1n1
```

If it's mounted somewhere, unmount first.

### Problem: `vgcreate` says "already exists in volume group X"

The PV is already part of another VG. Check:

```bash
sudo pvs
```

Either use the existing VG, or remove the PV from its current VG (only if safe):

```bash
sudo vgreduce <old-vg> /dev/nvme1n1
sudo pvremove /dev/nvme1n1
```

### Problem: machine fails to boot after fstab change

1. Boot into recovery mode (AWS: use serial console, or attach the root volume to another instance).
2. Comment out the `/data` line in `/etc/fstab`.
3. Reboot normally.
4. Fix the underlying issue (usually wrong UUID or missing device).

> 💡 **Prevention:** Always test with `mount -a` **before** rebooting — that's the whole point of Step 9.

### Problem: `resize2fs` says "nothing to do"

The LV wasn't extended first, or the filesystem already matches. Check with:

```bash
sudo lvs
df -h /data
```

If sizes match, nothing to do. If the LV is bigger but `df` doesn't reflect it, run `resize2fs` again.

### Problem: after extending, `df` doesn't show the new size

You forgot the second step. Run:

```bash
sudo resize2fs /dev/vgk8s/lvroot
```

`lvextend` grows the **block device**. `resize2fs` grows the **filesystem**. Both are needed.

---

## Important production vs lab distinctions

| Setting | Lab choice | Production consideration |
|---------|-----------|--------------------------|
| LV size | 50 GB out of 60 GB | Size for expected image + pod state growth |
| VG free space | ~10 GB reserved | Leave room for etcd, logs, or future LVs |
| `lost+found` | Ignored | Normal — created by `mkfs.ext4` |
| `rm -rf /var/lib/containerd` | Safe on fresh node | **Dangerous on live node** — stop services first |
| Reboot test | Recommended | **Mandatory** before production handoff |
| Snapshots | Optional | Take before risky changes (resize, move) |
| fstab mount options | `defaults` | Consider `noatime`, `nodiratime` for kubelet dirs |
| Backup of `/etc/fstab` | Manual | Always `cp /etc/fstab /etc/fstab.bak` before edits |

### Optional: add `noatime` for performance

`atime` updates trigger extra disk writes. For a busy Kubernetes node, disabling it can help:

```bash
# Edit the fstab entry to:
UUID=<uuid> /data ext4 defaults,noatime 0 2
```

Then:

```bash
sudo mount -o remount /data
findmnt /data
```

---

## Where to apply this

This exact procedure should be repeated on **every Kubernetes node** that has an extra disk — control planes (cp01, cp02, cp03) and workers (worker1, worker2).

Each node gets its own LVM setup. The UUIDs will be different per node — that's expected.

**Per-node checklist:**

- [ ] `lsblk` confirms the extra disk
- [ ] `pvcreate` on the extra disk
- [ ] `vgcreate vgk8s`
- [ ] `lvcreate lvroot` (50 G, or sized to fit)
- [ ] `mkfs.ext4` on the LV
- [ ] `/data` created and mounted
- [ ] fstab entry added and `mount -a` tested
- [ ] `/data/containerd` and `/data/kubelet` created
- [ ] Symlinks created
- [ ] Reboot tested
- [ ] `~10 GB` left free in the VG

---
