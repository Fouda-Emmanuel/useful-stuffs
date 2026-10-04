# HA Kubernetes Infrastructure — DNS (BIND9) + Load Balancer (HAProxy) Setup

**Domain:** `faekcorp.lab`
**Environment:** AWS EC2 (Ubuntu), private `10.70.x.x` network
**Goal:** Provide DNS resolution and a Kubernetes API load balancer for a High-Availability Kubernetes cluster.

---

## 📖 Before We Start 

**What you'll build:**

- A **DNS server** so machines can use names (`cp01.faekcorp.lab`) instead of IPs (`10.70.21.6`).
- A **load balancer** so Kubernetes clients hit one address (`haproxy.faekcorp.lab:6443`) and get routed to a healthy control plane.


**Prerequisites:**

- SSH access to the `haproxy` node with `sudo`.
- The `haproxy` node has a static private IP: `10.70.11.192`.
- You know the IPs of your Kubernetes nodes (given below).

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Server Inventory](#2-server-inventory)
3. [Part A — BIND9 DNS Server Setup](#part-a--bind9-dns-server-setup)
4. [Part B — System Resolver (systemd-resolved)](#part-b--system-resolver-systemd-resolved)
5. [Part C — HAProxy for Kubernetes API](#part-c--haproxy-for-kubernetes-api)
6. [Verification & Testing](#6-verification--testing)
7. [Troubleshooting Log](#7-troubleshooting-log)
8. [Key Concepts Explained](#8-key-concepts-explained)
9. [Glossary](#9-glossary)

---

## 1. Architecture Overview

Before touching any config, understand **what we're building and why**.

```
                         Internet DNS
                       1.1.1.1 / 8.8.8.8
                              ▲
                              │ forward unknown queries
                              │
                    ┌─────────────────────┐
                    │       HAProxy       │
                    │   10.70.11.192      │
                    │ haproxy.faekcorp.lab│
                    │                     │
                    │  ┌───────────────┐  │
                    │  │  BIND9 :53    │  │  ← answers DNS questions
                    │  │  authoritative│  │
                    │  │  faekcorp.lab │  │
                    │  └───────────────┘  │
                    │  ┌───────────────┐  │
                    │  │  HAProxy      │  │  ← balances Kubernetes API traffic
                    │  │  :6443 → k8s  │  │
                    │  │  :9999 stats  │  │
                    │  └───────────────┘  │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
        cp01                 cp02                 cp03
    10.70.21.6          10.70.31.209         10.70.41.250
          │
     ┌────┴────┐
   worker1   worker2
 10.70.61.85  10.70.11.67
```

### Why do we need this?

Imagine your Kubernetes nodes only had IPs:

```
cp01 = 10.70.21.6
cp02 = 10.70.31.209
cp03 = 10.70.41.250
```

You'd have to remember each one. Worse, your `kubeconfig` would point to one specific node — and if that node died, your whole cluster would be unreachable.

**Solution 1 — Names (DNS):**
Use `cp01.faekcorp.lab` instead of `10.70.21.6`. Easier for humans, stable if IPs ever change.

**Solution 2 — One address for the API (HAProxy):**
Point `kubectl` at `haproxy.faekcorp.lab:6443`. HAProxy quietly forwards each request to a healthy control plane. If `cp02` dies, HAProxy stops sending traffic to it — `kubectl` never notices.

### The two roles on the `haproxy` node

| Role | Software | Port | What it does |
|------|----------|------|--------------|
| DNS | BIND9 | `53` | Answers name → IP queries for `faekcorp.lab` |
| Load balancer | HAProxy | `6443` | Routes Kubernetes API traffic to cp01/cp02/cp03 |
| Stats page | HAProxy | `9999` | Web UI showing backend health |

> ⚠️ **Common misconception:** HAProxy and BIND9 are **separate services** on the same machine. They don't depend on each other. You could run them on different servers if you wanted.

---

## 2. Server Inventory

Keep this table handy — you'll reference it throughout.

| Hostname | FQDN | IP | Role |
|----------|------|-----|------|
| haproxy | `haproxy.faekcorp.lab` | `10.70.11.192` | DNS + API load balancer |
| cp01 | `cp01.faekcorp.lab` | `10.70.21.6` | K8s control plane |
| cp02 | `cp02.faekcorp.lab` | `10.70.31.209` | K8s control plane |
| cp03 | `cp03.faekcorp.lab` | `10.70.41.250` | K8s control plane |
| worker1 | `worker1.faekcorp.lab` | `10.70.61.85` | K8s worker |
| worker2 | `worker2.faekcorp.lab` | `10.70.11.67` | K8s worker |

> 🧠 **FQDN** = Fully Qualified Domain Name. `cp01` is a hostname. `cp01.faekcorp.lab` is the FQDN.

---

## Part A — BIND9 DNS Server Setup

> 🧠 **What is BIND9?** BIND (Berkeley Internet Name Domain) is the most widely-used DNS server software on the planet. Think of it as the program that answers the question *"what IP belongs to this name?"*.

All commands run on `haproxy` (`10.70.11.192`).

### A.1 Install BIND9

```bash
sudo apt update
sudo apt install -y bind9 bind9-utils dnsutils
```

**What each package does:**

| Package | Purpose | Why you need it |
|---------|---------|-----------------|
| `bind9` | The DNS server itself (daemon is called `named`) | Does the actual work |
| `bind9-utils` | Tools: `named-checkconf`, `named-checkzone` | To **validate configs before restarting** — critical |
| `dnsutils` | Tools: `dig`, `nslookup`, `host` | To **test DNS queries** |

> 💡 **Tip :** `apt update` refreshes the list of available packages. It does **not** upgrade anything. You must run it before installing new packages.

### A.2 Global options — `/etc/bind/named.conf.options`

This file tells BIND **how to behave** in general — not which zones it owns.

```bash
sudo tee /etc/bind/named.conf.options >/dev/null <<'EOF'
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { any; };
    allow-query { any; };

    listen-on { any; };
    listen-on-v6 { none; };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    dnssec-validation no;
};
EOF
```

> 🧠 **What's `tee <<'EOF'`?** It's a shell trick to write a multi-line block into a file in one go. Everything between `<<'EOF'` and the final `EOF` gets written to the file. The `>'/dev/null'` just hides the duplicated output. You could do the same with `sudo nano /etc/bind/named.conf.options` — but this is reproducible for scripts.

**Explain each line:**

| Directive | Meaning | Why it matters |
|-----------|---------|----------------|
| `directory "/var/cache/bind";` | Where BIND keeps its cache | Default; don't change unless you know why |
| `recursion yes;` | BIND may resolve names it isn't authoritative for | Needed so it can look up `google.com` etc. |
| `allow-recursion { any; };` | Anyone can ask BIND to recurse | ⚠️ Lab-only. Production: restrict to your subnet |
| `allow-query { any; };` | Anyone can query BIND | ⚠️ Lab-only. Production: restrict |
| `listen-on { any; };` | Listen on all IPv4 interfaces | So other nodes can reach it on `:53` |
| `listen-on-v6 { none; };` | Disable IPv6 | Lab uses IPv4 only — simpler |
| `forwarders { ... };` | Where to send unknown queries | Cloudflare + Google public DNS |
| `dnssec-validation no;` | Skip DNSSEC validation | Simplifies lab; production would enable |

> ⚠️ **Production warning:** `allow-query { any; }` and `allow-recursion { any; }` are **open resolvers**. On the public internet that's an abuse vector (DNS amplification attacks). In production, restrict to trusted networks like `10.70.0.0/16`.

### A.3 Declare the zone — `/etc/bind/named.conf.local`

This file tells BIND: **"You are the authority for `faekcorp.lab`."**

```bash
sudo tee /etc/bind/named.conf.local >/dev/null <<'EOF'
zone "faekcorp.lab" {
    type master;
    file "/etc/bind/db.faekcorp.lab";
};
EOF
```

> 🧠 **What's a "zone"?** A zone is a chunk of the DNS namespace that one server is responsible for. Here, `faekcorp.lab` is the zone, and BIND is its "master" (a.k.a. primary).

- `type master;` → this server holds the **authoritative** copy of the zone. (Modern BIND also accepts `type primary;` — same meaning.)
- `file` → where the zone data lives on disk.

### A.4 The zone file — `/etc/bind/db.faekcorp.lab`

This is where the **actual name → IP mappings** live.

```bash
sudo tee /etc/bind/db.faekcorp.lab >/dev/null <<'EOF'
$TTL 86400
@   IN  SOA haproxy.faekcorp.lab. admin.faekcorp.lab. (
        2025121701  ; Serial  (increment on change)
        3600        ; Refresh (1h)
        1800        ; Retry   (30m)
        604800      ; Expire  (7d)
        86400       ; Negative cache TTL (1d)
)

; Name server
@       IN  NS      haproxy.faekcorp.lab.

; A records
haproxy IN  A       10.70.11.192
cp01    IN  A       10.70.21.6
cp02    IN  A       10.70.31.209
cp03    IN  A       10.70.41.250
worker1 IN  A       10.70.61.85
worker2 IN  A       10.70.11.67
EOF
```

**Let's decode this line by line.**

#### `$TTL 86400`

Time To Live = 86400 seconds = **1 day**.
Tells DNS clients: *"You can cache this answer for up to 1 day before asking again."*

#### The SOA record

SOA = **Start of Authority**. Every zone must have exactly one. It describes **who's in charge and how the zone behaves**.

```
@   IN  SOA haproxy.faekcorp.lab. admin.faekcorp.lab. (
```

- `@` = shorthand for the zone name → `faekcorp.lab.`
- `haproxy.faekcorp.lab.` = the primary name server for this zone
- `admin.faekcorp.lab.` = the admin's email **in DNS format** (the `@` in an email becomes a dot)

> 🧠 **Why `admin.faekcorp.lab.` instead of `admin@faekcorp.lab`?** Because `@` already has a special meaning in zone files. So DNS uses a dot instead. Yes, this confuses everyone at first.

The five numbers in parentheses:

| # | Value | Meaning | Human-readable |
|---|-------|---------|----------------|
| 1 | `2025121701` | Serial | Zone version (see below) |
| 2 | `3600` | Refresh | Secondaries re-check primary every 1h |
| 3 | `1800` | Retry | If check fails, retry every 30m |
| 4 | `604800` | Expire | If no contact for 7d, secondaries stop answering |
| 5 | `86400` | Negative TTL | Cache NXDOMAIN answers for 1 day |

> 🧠 **Serial number rule:** Whenever you edit the zone, **increment the serial**. Secondaries compare serials to know if the zone changed. Format `YYYYMMDDNN` is a common convention (year/month/day + counter).

#### The NS record

```
@       IN  NS      haproxy.faekcorp.lab.
```

NS = **Name Server**. Says: *"The DNS server for `faekcorp.lab` is `haproxy.faekcorp.lab`."*

#### The A records

```
cp01    IN  A       10.70.21.6
```

A = **Address** record. Maps a name to an IPv4 address.

**Important:** Notice the names are written **short** (`cp01`), not as `cp01.faekcorp.lab`. That's because we're already inside the zone — BIND automatically appends `.faekcorp.lab` to any name that doesn't end in a dot.

> ⚠️ **Trailing-dot rule:**
> - `haproxy.faekcorp.lab.` (with trailing dot) = **exactly** this FQDN
> - `haproxy.faekcorp.lab` (no dot) = relative → BIND appends zone → becomes `haproxy.faekcorp.lab.faekcorp.lab` ❌
> - Always use the trailing dot on the right-hand side of SOA/NS records.

### A.5 Fix permissions

```bash
sudo chown root:bind /etc/bind/db.faekcorp.lab
sudo chmod 640 /etc/bind/db.faekcorp.lab
```

| Setting | Result |
|---------|--------|
| `chown root:bind` | File owned by `root`, group `bind` |
| `chmod 640` | `root`: read+write · `bind` group: read · others: nothing |

BIND runs as the `bind` user. It needs **read** access. Nobody else does.

### A.6 Validate BEFORE starting

> 🧠 **Golden rule:** Never restart a service with an invalid config in production. Always validate first.

```bash
sudo named-checkconf
sudo named-checkzone faekcorp.lab /etc/bind/db.faekcorp.lab
```

- `named-checkconf` → validates **configuration files** (`named.conf.*`). No output = success.
- `named-checkzone` → validates the **zone file**. Success looks like:

```
zone faekcorp.lab/IN: loaded serial 2025121701
OK
```

> 💡 If `named-checkzone` prints `OK`, BIND will accept the zone. If not, it tells you exactly which line is broken.

### A.7 Enable and start BIND

```bash
sudo systemctl enable --now named
sudo systemctl status named --no-pager
```

> 🧠 **`enable --now`** = two things at once:
> - `enable` → start automatically at boot
> - `--now` → also start it right now
> Same as running `systemctl enable named && systemctl start named`.

Look for this in the status output:

```
Active: active (running)
```

> ⚠️ **A running service isn't necessarily a working service.** That's why we validated the configs first, and why we test next.

### A.8 Test BIND directly

```bash
dig +short cp01.faekcorp.lab @127.0.0.1
dig +short haproxy.faekcorp.lab @127.0.0.1
dig +short worker1.faekcorp.lab @127.0.0.1
```

**Break down the command:**

- `dig` = DNS lookup tool
- `+short` = only show the answer, no fluff
- `@127.0.0.1` = ask **BIND directly** (bypasses the OS resolver)

Expected output:

```
10.70.21.6
10.70.11.192
10.70.61.85
```

At this point: **BIND is working.** But applications can't use it yet — the OS isn't telling them to.

---

## Part B — System Resolver (systemd-resolved)

### B.1 Why this step?

`dig @127.0.0.1` explicitly says *"use BIND"*. But normal commands don't do that:

```bash
ping cp01.faekcorp.lab
```

This uses the **system resolver** — a separate component that decides which DNS server to ask. On Ubuntu that's **systemd-resolved**.

Before this step, systemd-resolved doesn't know about our BIND server. So `ping` fails with `Name or service not known`, even though `dig @127.0.0.1` works.

### B.2 Configure `/etc/systemd/resolved.conf`

```bash
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=faekcorp.lab
DNSSEC=no
FallbackDNS=
EOF
```

| Directive | Meaning | Why |
|-----------|---------|-----|
| `DNS=127.0.0.1` | Use the DNS server on localhost | BIND runs here |
| `Domains=faekcorp.lab` | Mark this as a routing/search domain | Makes `ping cp01` (short name) work too |
| `DNSSEC=no` | Skip DNSSEC validation | Matches BIND config |
| `FallbackDNS=` | No fallback | BIND forwards upstream itself |

> 🧠 **Why `127.0.0.1` instead of `10.70.11.192`?**
> Because BIND is running on **this same machine**. `127.0.0.1` (loopback) is the shortest, most reliable path — it works even if the network interface is down. Other nodes query `10.70.11.192:53`; this machine queries itself via loopback.

### B.3 Restart and verify

```bash
sudo systemctl restart systemd-resolved
resolvectl status | sed -n '1,20p'
```

Look for:

```
DNS Servers: 127.0.0.1
DNS Domain: faekcorp.lab
```

### B.4 Test through the system resolver

```bash
getent hosts cp01.faekcorp.lab
```

> 🧠 **What is `getent`?** It uses the **same resolution path** your applications use (`ping`, `curl`, `ssh`). If `getent` works, everything works.

Expected:

```
10.70.21.6      cp01.faekcorp.lab
```

### B.5 The complete name-resolution flow

Now that everything is wired up, this is what happens when you run `ping cp01.faekcorp.lab`:

```
Application (ping)
        ↓
/etc/resolv.conf  →  nameserver 127.0.0.53   (systemd-resolved stub)
        ↓
systemd-resolved
        ↓
127.0.0.1:53   (BIND9)
        ↓
├── faekcorp.lab → answered locally from zone file
└── everything else → forwarded to 1.1.1.1 / 8.8.8.8
```

> 🧠 **Why `127.0.0.53`?** That's a stub listener systemd-resolved runs. Applications point to it via `/etc/resolv.conf`; systemd-resolved then decides the real DNS server. You don't configure this — it's automatic.

---

## Part C — HAProxy for Kubernetes API

### C.1 Install HAProxy

```bash
sudo apt install -y haproxy
```

> 🧠 **What is HAProxy?** A high-performance TCP/HTTP load balancer. In our case it acts as a **TCP proxy**: it accepts connections on `:6443` and forwards them to one of the healthy control planes.

### C.2 Configuration — `/etc/haproxy/haproxy.cfg`

```haproxy
# =========================================================
# HAProxy configuration for Kubernetes API
# =========================================================

global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon
    maxconn 10000
    pidfile /var/run/haproxy.pid


# ---------------------------------------------------------
# Default settings
# ---------------------------------------------------------

defaults
    log global
    mode tcp
    option tcplog

    timeout connect 10s
    timeout client 1m
    timeout server 1m
    timeout check 5s

    retries 3


# ---------------------------------------------------------
# DNS resolvers
# BIND9 is running on the same machine as HAProxy
# ---------------------------------------------------------

resolvers faekcorp_dns
    nameserver dns1 10.70.11.192:53
    resolve_retries 3
    timeout retry 1s
    hold valid 10s


# ---------------------------------------------------------
# Frontend - Kubernetes API
# ---------------------------------------------------------

frontend k8s_api
    bind 10.70.11.192:6443
    mode tcp
    default_backend k8s_api_backend


# ---------------------------------------------------------
# Backend - Kubernetes Control Plane
# ---------------------------------------------------------

backend k8s_api_backend
    mode tcp
    balance roundrobin

    option tcp-check
    tcp-check connect port 6443

    default-server inter 3s fall 3 rise 2

    server cp01 10.70.21.6:6443 check
    server cp02 10.70.31.209:6443 check
    server cp03 10.70.41.250:6443 check


# ---------------------------------------------------------
# HAProxy Statistics
# ---------------------------------------------------------

listen stats
    bind *:9999
    mode http
    stats enable
    stats uri /stats
    stats hide-version
    stats refresh 5s
```

**Section by section:**

#### `global`

Process-wide settings. `daemon` = run in background. `maxconn 10000` = max simultaneous connections.

#### `defaults`

Settings inherited by all frontends/backends. `mode tcp` = we're proxying raw TCP (Kubernetes API uses TLS over TCP — HAProxy doesn't need to inspect HTTP).

#### `resolvers faekcorp_dns`

> 🧠 **This block is only needed if you specify backends by hostname** (e.g. `server cp01 cp01.faekcorp.lab:6443`). Since we're using IPs, this block isn't actually used — but it's good to have ready.
>
> The name `faekcorp_dns` is an **arbitrary label** — you could call it `dns_training`, `my_dns`, `cluster_dns` — anything.

#### `frontend k8s_api`

The **listening side**. `bind 10.70.11.192:6443` = accept connections on that address+port.

#### `backend k8s_api_backend`

The **server pool**. Key directives:

| Directive | Meaning |
|-----------|---------|
| `balance roundrobin` | Rotate through servers in order |
| `option tcp-check` | Health check = open a TCP connection |
| `tcp-check connect port 6443` | Test port 6443 specifically |
| `default-server inter 3s fall 3 rise 2` | Check every 3s; 3 fails = DOWN; 2 OK = UP |
| `server cp01 10.70.21.6:6443 check` | Define a backend + enable health checks |

> 🧠 **What does `check` do?** Without it, HAProxy blindly sends traffic to every server. With it, HAProxy probes each server. Failing servers are removed from rotation until they recover.

**Health check lifecycle:**

```
Every 3 seconds:
    HAProxy → cp01:6443  ✅
    HAProxy → cp02:6443  ✅
    HAProxy → cp03:6443  ✅

cp02 crashes:
    HAProxy → cp02:6443  ❌  (1)
    HAProxy → cp02:6443  ❌  (2)
    HAProxy → cp02:6443  ❌  (3)  ← fall 3 reached
    → cp02 removed from rotation

cp02 recovers:
    HAProxy → cp02:6443  ✅  (1)
    HAProxy → cp02:6443  ✅  (2)  ← rise 2 reached
    → cp02 back in rotation
```

**Client experience:** Always connects to `haproxy:6443`. Never notices cp02 went down.

#### `listen stats`

Web UI on `:9999` for monitoring. Visit `http://10.70.11.192:9999/stats` in a browser.

### C.3 Validate before starting

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Expected: `Configuration file is valid`

> 🧠 `-c` = check only, don't start. `-f` = config file path.

### C.4 Enable and start

```bash
sudo systemctl enable --now haproxy
sudo systemctl status haproxy --no-pager
```

### C.5 Check listening ports

```bash
sudo ss -tlnp | grep -E ':(53|6443|9999)'
```

You should see BIND on `:53`, HAProxy on `:6443`, and stats on `:9999`.

---

## 6. Verification & Testing

### DNS layer

```bash
# Direct to BIND
dig +short cp01.faekcorp.lab @127.0.0.1

# Through system resolver
getent hosts cp01.faekcorp.lab

# Full dig output (shows TTL, authority, etc.)
dig cp01.faekcorp.lab @127.0.0.1
```

### HAProxy layer

```bash
# Config validity
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Listening ports
sudo ss -tlnp | grep -E ':(53|6443|9999)'

# TCP reachability to the LB
nc -vz 10.70.11.192 6443
```

### Full chain

```bash
getent hosts haproxy.faekcorp.lab
getent hosts cp01.faekcorp.lab
getent hosts worker1.faekcorp.lab
```

All three should return the correct IP.

---

## 7. Troubleshooting Log

### Issue 1 — `ping: cp01.faekcorp.lab: Name or service not known`

**Symptom:** `dig @127.0.0.1` works but `ping cp01.faekcorp.lab` fails.

**Cause:** `systemd-resolved` isn't pointing at BIND.

**Fix:**

```bash
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=faekcorp.lab
DNSSEC=no
FallbackDNS=
EOF
sudo systemctl restart systemd-resolved
```

**Verify:** `getent hosts cp01.faekcorp.lab` returns `10.70.21.6`.

---

### Issue 2 — `ping cp01.faekcorp.lab` → 100% packet loss

**Symptom:**

```
PING cp01.faekcorp.lab (10.70.21.6) 56(84) bytes of data.
--- cp01.faekcorp.lab ping statistics ---
4 packets transmitted, 0 received, 100% packet loss
```

**Analysis:** DNS **worked** — `ping` resolved the name to `10.70.21.6`. The failure is ICMP, not DNS.

**Cause:** AWS Security Groups typically block ICMP by default. **This does not mean Kubernetes is broken** — TCP to `:6443` may still work fine.

> 🧠 **Lesson:** `ping` uses **ICMP**. `nc -vz host 6443` uses **TCP**. Always test the protocol/port you actually care about.

**Next steps:**

```bash
ip route get 10.70.21.6
nc -vz 10.70.21.6 6443
nc -vz 10.70.21.6 22
```

If TCP 6443 connects but ping fails → routing is fine for the services that matter.

**Fix (if you actually need ICMP):** Add an inbound rule in the `cp01` Security Group allowing **All ICMP – IPv4** from the `haproxy` security group (or from `10.70.0.0/16`).

---

### Issue 3 — Confusion: `DNS=127.0.0.1` vs `DNS=10.70.11.192`

**Question:** Why use `127.0.0.1` when the machine's real IP is `10.70.11.192`?

**Answer:** BIND runs **on this machine**. Loopback is the shortest, most reliable path — no NIC, no subnet dependency. Other nodes query `10.70.11.192:53`; the local machine queries itself via `127.0.0.1`.

---

### Issue 4 — `resolvers dns_training` — is this name required?

**Answer:** No. It's an **arbitrary label**. It only matters if backend servers are specified by hostname. With IP-based backends it's unused.

---

### Issue 5 — Config valid but service won't start

```bash
sudo journalctl -u named -n 50 --no-pager
sudo journalctl -u haproxy -n 50 --no-pager
```

Look for errors. Common causes:

- Zone file permissions wrong (BIND can't read it).
- Port 53 already in use (`systemd-resolved` listening on `127.0.0.53:53` is fine; something else on `:53` may conflict).
- Syntax error not caught by `named-checkconf` (rare).

---

## 8. Key Concepts Explained

### Authoritative vs Recursive DNS

| | Authoritative | Recursive |
|---|---|---|
| Knows | Its own zone (`faekcorp.lab`) | Nothing by default |
| Role | Answers from its zone data | Looks up names on behalf of clients |
| Example | `cp01.faekcorp.lab` → `10.70.21.6` | `google.com` → forwards to 1.1.1.1 |

Our BIND server does **both**.

### Forwarders

BIND only knows `faekcorp.lab`. For anything else (`github.com`, `ubuntu.com`), it forwards to `1.1.1.1` / `8.8.8.8`.

### `@` in zone files

`@` = the zone origin = `faekcorp.lab.` So `@ IN NS haproxy.faekcorp.lab.` declares the NS record at the zone root.

### Trailing dots

- `haproxy.faekcorp.lab.` (with dot) = **fully qualified**, exactly as written.
- `haproxy.faekcorp.lab` (no dot) = **relative**, gets `.faekcorp.lab` appended → wrong.

### `type master` vs `type primary`

Identical meaning. `master` is the older term; `primary` is preferred in modern BIND. Both work.

### Round-robin vs health-checked load balancing

- **Plain round-robin** → blindly rotates across all servers.
- **With `check`** → HAProxy probes each backend and removes failing ones until they recover.

### TCP check vs HTTP check

`option tcp-check` + `tcp-check connect port 6443` → HAProxy only confirms **TCP can connect**. It doesn't verify Kubernetes API health. For this course, if `kube-apiserver` is listening, the node is considered healthy.

---

## 9. Glossary

| Term | Meaning |
|------|---------|
| **A record** | Maps a name → IPv4 address |
| **AAAA record** | Maps a name → IPv6 address |
| **BIND** | The most common DNS server software |
| **DNS** | Domain Name System — the internet's "phone book" |
| **FQDN** | Fully Qualified Domain Name, e.g. `cp01.faekcorp.lab` |
| **Forwarder** | An upstream DNS server BIND forwards unknown queries to |
| **HAProxy** | A TCP/HTTP load balancer |
| **ICMP** | Protocol used by `ping` — often blocked by cloud firewalls |
| **named** | The name of the BIND daemon process |
| **NS record** | Declares which server is the name server for a zone |
| **NXDOMAIN** | DNS response meaning "this name doesn't exist" |
| **Recursion** | Looking up names on behalf of a client |
| **Resolver** | A component that turns names into IPs |
| **Round-robin** | Load-balancing strategy that rotates through servers |
| **SOA record** | Start of Authority — mandatory first record in a zone |
| **TTL** | Time To Live — how long to cache a DNS answer |
| **Zone** | A chunk of the DNS namespace a server is authoritative for |
| **Zone file** | Text file containing all records for a zone |

---

## Appendix — Complete Setup Sequence (cheat sheet)

> Copy-paste friendly. For understanding, read the full sections above.

```bash
# ─── 1. Install ───────────────────────────────────────────
sudo apt update
sudo apt install -y bind9 bind9-utils dnsutils haproxy

# ─── 2. BIND global options ──────────────────────────────
sudo tee /etc/bind/named.conf.options >/dev/null <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-recursion { any; };
    allow-query { any; };
    listen-on { any; };
    listen-on-v6 { none; };
    forwarders { 1.1.1.1; 8.8.8.8; };
    dnssec-validation no;
};
EOF

# ─── 3. BIND zone declaration ────────────────────────────
sudo tee /etc/bind/named.conf.local >/dev/null <<'EOF'
zone "faekcorp.lab" {
    type master;
    file "/etc/bind/db.faekcorp.lab";
};
EOF

# ─── 4. Zone file ────────────────────────────────────────
sudo tee /etc/bind/db.faekcorp.lab >/dev/null <<'EOF'
$TTL 86400
@   IN  SOA haproxy.faekcorp.lab. admin.faekcorp.lab. (
        2025121701 3600 1800 604800 86400 )
@       IN  NS      haproxy.faekcorp.lab.
haproxy IN  A       10.70.11.192
cp01    IN  A       10.70.21.6
cp02    IN  A       10.70.31.209
cp03    IN  A       10.70.41.250
worker1 IN  A       10.70.61.85
worker2 IN  A       10.70.11.67
EOF

# ─── 5. Permissions + validate + start ──────────────────
sudo chown root:bind /etc/bind/db.faekcorp.lab
sudo chmod 640 /etc/bind/db.faekcorp.lab
sudo named-checkconf
sudo named-checkzone faekcorp.lab /etc/bind/db.faekcorp.lab
sudo systemctl enable --now named

# ─── 6. System resolver ──────────────────────────────────
sudo tee /etc/systemd/resolved.conf >/dev/null <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=faekcorp.lab
DNSSEC=no
FallbackDNS=
EOF
sudo systemctl restart systemd-resolved

# ─── 7. HAProxy config ───────────────────────────────────
sudo tee /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon
    maxconn 10000
    pidfile /var/run/haproxy.pid

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    timeout check 5s
    retries 3

resolvers faekcorp_dns
    nameserver dns1 10.70.11.192:53
    resolve_retries 3
    timeout retry 1s
    hold valid 10s

frontend k8s_api
    bind 10.70.11.192:6443
    mode tcp
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    tcp-check connect port 6443
    default-server inter 3s fall 3 rise 2
    server cp01 10.70.21.6:6443 check
    server cp02 10.70.31.209:6443 check
    server cp03 10.70.41.250:6443 check

listen stats
    bind *:9999
    mode http
    stats enable
    stats uri /stats
    stats hide-version
    stats refresh 5s
EOF

# ─── 8. Validate + start HAProxy ─────────────────────────
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy

# ─── 9. Verify ───────────────────────────────────────────
dig +short cp01.faekcorp.lab @127.0.0.1
getent hosts cp01.faekcorp.lab
sudo ss -tlnp | grep -E ':(53|6443|9999)'
nc -vz 10.70.11.192 6443
```

---
