# cAdvisor Crash / Restart Loop Due to Linux Inotify & Ulimit Limits

## 🧠 Overview

This document explains a real issue encountered when running a Docker observability stack (Prometheus + Grafana + node-exporter + cAdvisor) on a Linux development machine.

cAdvisor failed to start or entered a restart loop due to Linux kernel resource limits being too low.

This issue is also relevant in Kubernetes environments.

---

## Symptoms

### 1. cAdvisor container crashes or restarts

```bash
docker ps -a
````

Shows:

* `Restarting (255)`
* or `Exited`

---

### 2. Logs show this error

```text
inotify_init: too many open files
Failed to start manager
```

or

```text
Failed to start manager: inotify_init: too many open files
```

---

## Root Cause

The issue comes from **two Linux limits**:

---

### 1. inotify limits (filesystem watchers)

Check:

```bash
cat /proc/sys/fs/inotify/max_user_instances
cat /proc/sys/fs/inotify/max_user_watches
```

Default values are often too low:

* max_user_instances = 128 ❌
* max_user_watches = 65536 ⚠️ borderline

cAdvisor needs many watchers to monitor:

* containers
* filesystem changes
* processes

---

### 2. file descriptor limit (ulimit)

Check:

```bash
ulimit -n
```

Default:

* 1024 ❌ too low

cAdvisor + Docker requires many open files/sockets.

---

## Diagnosis Steps

```bash
docker logs cadvisor
```

Look for:

```text
inotify_init: too many open files
```

or

```text
Failed to start manager
```

---

## Temporary Fix (safe for testing)

### Step 1 — Increase inotify limits

```bash
sudo sysctl -w fs.inotify.max_user_instances=1024
sudo sysctl -w fs.inotify.max_user_watches=524288
```

---

### Step 2 — Increase file descriptor limit (current session only)

```bash
ulimit -n 65536
```

---

### Step 3 — Restart cAdvisor

```bash
docker restart cadvisor
```

---

## Expected Result After Fix

```bash
docker ps
```

cAdvisor should show:

```
Up (healthy)
```

And logs:

```
Listening on :8080
Starting cadvisor manager
```

---

## Permanent Fix

### 1. Persist inotify settings

```bash
sudo nano /etc/sysctl.conf
```

Add:

```conf
fs.inotify.max_user_instances=1024
fs.inotify.max_user_watches=524288
```

Apply:

```bash
sudo sysctl -p
```

---

### 2. Persist file descriptor limit

```bash
sudo nano /etc/security/limits.conf
```

Add:

```conf
* soft nofile 65536
* hard nofile 65536
```

---

## Reset to Default (IMPORTANT)

If you want to revert system changes:

### Reset inotify values

```bash
sudo sysctl -w fs.inotify.max_user_instances=128
sudo sysctl -w fs.inotify.max_user_watches=65536
```

---

### Reset file descriptor limit

```bash
ulimit -n 1024
```

---

## Notes

* These changes do NOT affect CPU or memory
* They only increase monitoring capacity limits
* Safe for development machines
* Required for tools like:

  * cAdvisor
  * Prometheus exporters
  * Kubernetes nodes

---

## Key Learning

This issue is not a Docker bug.

It is a Linux kernel limitation exposed by observability workloads.

---

# Kubernetes Relevance (VERY IMPORTANT)

This issue is not specific to Docker. It also appears in Kubernetes because:

* cAdvisor is integrated into kubelet (historically and conceptually)
* each node runs many containers/pods
* monitoring agents increase system resource usage

---

## Why this matters in Kubernetes

In Kubernetes nodes:

* each pod increases filesystem watches
* kubelet + container runtime increases open files usage
* monitoring agents amplify load

---

## Kubernetes Symptoms

* kubelet instability
* missing container metrics
* Prometheus scraping failures
* node exporter gaps
* cAdvisor metrics missing

---

## Check limits on a Kubernetes node

### File descriptor limit

```bash
cat /proc/$(pidof kubelet)/limits | grep "open files"
```

or:

```bash
ulimit -n
```

---

### Inotify limits

```bash
cat /proc/sys/fs/inotify/max_user_instances
cat /proc/sys/fs/inotify/max_user_watches
```

---

## Kubernetes Node Fix

### Temporary fix

```bash
sudo sysctl -w fs.inotify.max_user_instances=1024
sudo sysctl -w fs.inotify.max_user_watches=524288
```

---

### Persistent fix

```bash
sudo nano /etc/sysctl.conf
```

Add:

```conf
fs.inotify.max_user_instances=1024
fs.inotify.max_user_watches=524288
```

Apply:

```bash
sudo sysctl -p
```

---

## Kubernetes ulimit fix (systemd kubelet)

```bash
sudo systemctl edit kubelet
```

Add:

```ini
[Service]
LimitNOFILE=65536
```

Then:

```bash
sudo systemctl daemon-reexec
sudo systemctl restart kubelet
```

---

## Why Kubernetes needs higher limits

A single node may run:

* 50–200 pods
* multiple containers per pod
* sidecars (logging, monitoring, service mesh)

Each increases:

* file descriptors
* inotify watchers
* network sockets

---

## Key takeaway

This issue often appears when moving from:

Docker (small scale) → Kubernetes (large scale)

---

## Mental model

| System              | Load      |
| ------------------- | --------- |
| Docker              | moderate  |
| Docker + monitoring | high      |
| Kubernetes node     | very high |

---

## Final conclusion

This issue is caused by Linux kernel resource limits being too low for observability workloads.

Fixing it requires tuning:

* inotify limits
* file descriptor limits

Not Docker itself.
