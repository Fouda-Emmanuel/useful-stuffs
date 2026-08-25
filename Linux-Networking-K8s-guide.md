# Linux & Kubernetes Network Troubleshooting Guide

## Complete Reference with Tool Explanations, Commands, and Real-World Examples

A practical guide for system administrators, DevOps engineers, and Kubernetes professionals.

---

## Table of Contents

1. [Introduction & Mental Model](#1-introduction--mental-model)
2. [Core Networking Tools](#2-core-networking-tools)
3. [Interface Inspection](#3-interface-inspection)
4. [IP Addressing](#4-ip-addressing)
5. [MAC Addresses](#5-mac-addresses)
6. [Routing](#6-routing)
7. [Bridges and Virtual Networking](#7-bridges-and-virtual-networking)
8. [Process Management](#8-process-management)
9. [Socket Inspection (ss)](#9-socket-inspection-ss)
10. [Advanced Networking Tools](#10-advanced-networking-tools)
11. [Container & Kubernetes Networking](#11-container--kubernetes-networking)
12. [Complete Troubleshooting Workflows](#12-complete-troubleshooting-workflows)
13. [Quick Reference Cheat Sheet](#13-quick-reference-cheat-sheet)
14. [Common Ports Reference](#14-common-ports-reference)

---

## 1. Introduction & Mental Model

### The Linux Networking Stack

```
┌─────────────────────────────────────────────────────────────┐
│                      Application                           │
│  (curl, nc, kubectl, web server, database, etc.)           │
├─────────────────────────────────────────────────────────────┤
│                         Port                                │
│  (80, 443, 6443, 2379, 3306, etc.)                        │
├─────────────────────────────────────────────────────────────┤
│                     Socket (ss)                            │
│  (Established connections, listening ports)                 │
├─────────────────────────────────────────────────────────────┤
│                  Linux Network Stack                        │
│  (TCP/UDP handling, packet processing)                     │
├─────────────────────────────────────────────────────────────┤
│                   Routing Table (ip route)                 │
│  (Where packets should go)                                 │
├─────────────────────────────────────────────────────────────┤
│                Network Interface (ip link)                  │
│  (Physical: eth0, wlp2s0, enp0s31f6 | Virtual: docker0)    │
├─────────────────────────────────────────────────────────────┤
│                    MAC Address                              │
│  (Hardware/link layer address)                             │
├─────────────────────────────────────────────────────────────┤
│                    Physical Network                         │
│  (Wi-Fi, Ethernet, Overlay network)                        │
└─────────────────────────────────────────────────────────────┘
```

### Container/Kubernetes Networking Extension

```
┌─────────────────────────────────────────────────────────────┐
│                         Pod/Container                       │
│  (Has its own IP, network namespace)                       │
├─────────────────────────────────────────────────────────────┤
│                   veth (Virtual Ethernet)                   │
│  (Virtual cable connecting pod to bridge)                  │
├─────────────────────────────────────────────────────────────┤
│                    Linux Bridge                             │
│  (docker0, cni0, br-xxx - virtual switch)                  │
├─────────────────────────────────────────────────────────────┤
│                      CNI Plugin                             │
│  (Flannel, Calico, Cilium - overlay networking)            │
├─────────────────────────────────────────────────────────────┤
│                    Node Network                             │
│  (eth0, wlp2s0, enp0s31f6 - physical/virtual interface)   │
├─────────────────────────────────────────────────────────────┤
│              Another Node / External Network                │
└─────────────────────────────────────────────────────────────┘
```

### The Golden Question

> **"What question am I asking, and which command answers that question?"**

Don't memorize interface names. Don't assume `eth0` exists. Always inspect the actual system.

---

## 2. Core Networking Tools

### Tool: `ip`

**What it is:** The primary tool for Linux networking configuration and inspection. Replaces older tools like `ifconfig` and `route`.

**What it does:** Shows and configures network interfaces, IP addresses, routes, and more.

**Key subcommands:**
- `ip link` - Manage network interfaces
- `ip addr` - Manage IP addresses
- `ip route` - Manage routing tables
- `ip neigh` - Manage neighbor/ARP tables

**When to use:** For all basic network inspection tasks.

---

### Tool: `ss`

**What it is:** Socket statistics tool. Replaces `netstat` with better performance and more detailed output.

**What it does:** Shows information about network sockets (connections, listening ports, etc.)

**Common options:**
- `-l` - Show only listening sockets
- `-n` - Use numeric addresses (no DNS resolution)
- `-t` - Show TCP sockets only
- `-u` - Show UDP sockets only
- `-p` - Show process information
- `-a` - Show all sockets (listening and established)
- `-4` - IPv4 only
- `-6` - IPv6 only

**When to use:** Checking what ports are open, who is connected, debugging network services.

---

### Tool: `ps`

**What it is:** Process status command.

**What it does:** Shows running processes.

**Common options:**
- `ps -ef` - Full format listing with all processes
- `ps aux` - Detailed format with CPU/memory usage
- `ps -eo pid,comm,args` - Custom output format

**When to use:** Checking if services are running, finding process IDs, inspecting startup arguments.

---

### Tool: `bridge`

**What it is:** Tool for inspecting Linux bridges.

**What it does:** Shows bridge configuration and connected interfaces.

**Common use:** `bridge link` - shows interfaces connected to bridges.

**When to use:** Inspecting virtual network connections for containers and pods.

---

### Tool: `ping`

**What it is:** ICMP echo request/reply tool.

**What it does:** Tests basic IP connectivity using ICMP packets.

**When to use:** First-level connectivity testing.

**Important:** Success doesn't guarantee TCP/UDP service is working.

---

### Tool: `nc` (Netcat)

**What it is:** Network tool for TCP/UDP connections.

**What it does:** Can establish TCP connections, listen on ports, transfer data.

**Common options:**
- `-v` - Verbose
- `-z` - Zero I/O (just test connectivity)
- `-u` - UDP mode
- `-l` - Listen mode

**When to use:** Testing TCP/UDP connectivity to specific ports.

---

### Tool: `curl`

**What it is:** HTTP/HTTPS client.

**What it does:** Sends HTTP requests and displays responses.

**Common options:**
- `-k` - Ignore SSL certificate errors
- `-v` - Verbose
- `-I` - Headers only
- `-H` - Add header

**When to use:** Testing web services, APIs, Kubernetes API server.

---

### Tool: `tcpdump`

**What it is:** Packet capture tool.

**What it does:** Captures and displays network packets.

**When to use:** Deep packet inspection, debugging network traffic flow.

---

### Tool: `traceroute`

**What it is:** Route tracing tool.

**What it does:** Shows the path packets take to reach a destination.

**When to use:** Understanding network path, finding routing problems.

---

### Tool: `dig` / `nslookup`

**What they are:** DNS resolution tools.

**What they do:** Query DNS servers for domain name resolution.

**When to use:** DNS troubleshooting.

---

## 3. Interface Inspection

### `ip link` - List All Interfaces

**Purpose:** Show all network interfaces and their state.

```bash
ip link
```

**Sample Output:**
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000
    link/ether a4:4c:c8:00:b1:43 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether f8:59:71:a1:67:79 brd ff:ff:ff:ff:ff:ff
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 16:24:66:f7:4e:36 brd ff:ff:ff:ff:ff:ff
```

**Understanding Interface Flags:**
```
<BROADCAST,MULTICAST,UP,LOWER_UP>
```

| Flag | Meaning |
|------|---------|
| `UP` | Interface is administratively up |
| `LOWER_UP` | Physical layer is up (cable connected) |
| `BROADCAST` | Can broadcast packets |
| `MULTICAST` | Can multicast packets |
| `NO-CARRIER` | No physical connection detected |
| `state UP` | Interface is operational |
| `state DOWN` | Interface is not operational |
| `state UNKNOWN` | State cannot be determined (loopback) |

---

### `ip link show [interface]` - Specific Interface

**Purpose:** Show detailed information about one interface.

```bash
ip link show wlp2s0
```

**Sample Output:**
```
3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether f8:59:71:a1:67:79 brd ff:ff:ff:ff:ff:ff
```

**What We Learn:**
- Interface name: `wlp2s0`
- MAC address: `f8:59:71:a1:67:79`
- MTU: 1500 bytes
- State: UP (operational)

---

### `ip addr` - Show All IP Addresses

**Purpose:** Show interfaces with their IP configurations.

```bash
ip addr
```

**Sample Output:**
```
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever

2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> ...
    # No IP assigned, state DOWN

3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.1.172/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp2s0
       valid_lft 84898sec preferred_lft 84898sec
    inet6 fe80::a390:178d:d355:55f/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::1424:66ff:fef7:4e36/64 scope link
       valid_lft forever preferred_lft forever
```

**Understanding IP Flags:**
- `scope host` - Only local (127.0.0.1)
- `scope global` - Accessible externally
- `scope link` - Link-local only (IPv6)
- `dynamic` - DHCP assigned
- `static` - Manually configured

---

### `ip addr show [interface]` - Specific IP

**Purpose:** Show IP details for a specific interface.

```bash
ip addr show wlp2s0
```

**Sample Output:**
```
3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.1.172/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp2s0
    inet6 fe80::a390:178d:d355:55f/64 scope link noprefixroute
```

**What We Learn:**
- IPv4: `192.168.1.172`
- Subnet: `/24` (255.255.255.0)
- Broadcast: `192.168.1.255`
- DHCP assigned (`dynamic`)
- IPv6 link-local: `fe80::a390:178d:d355:55f/64`

---

## 4. IP Addressing

### Understanding CIDR Notation

CIDR notation defines the network mask:

| CIDR | Subnet Mask | Usable IPs |
|------|-------------|------------|
| `/8` | 255.0.0.0 | 16,777,214 |
| `/16` | 255.255.0.0 | 65,534 |
| `/24` | 255.255.255.0 | 254 |
| `/32` | 255.255.255.255 | 1 (single host) |

**Example:**
```
192.168.1.172/24
- Network: 192.168.1.0
- Host: 172
- Broadcast: 192.168.1.255
```

### Interface Types Explained

**Physical Interfaces:**
```
enp0s31f6    → Ethernet (wired)
wlp2s0       → Wi-Fi
eth0         → Common Ethernet (older naming)
ens33        → Ethernet (common in VMs)
```

**Virtual Interfaces:**
```
lo          → Loopback (127.0.0.1)
docker0     → Default Docker bridge
br-xxx      → Custom Docker bridge
cni0        → Kubernetes CNI bridge
flannel.1   → Flannel overlay
vethxxx     → Virtual Ethernet (container/pod)
tunl0       → TUN/TUNNEL interface
bond0       → Bonding interface
```

---

## 5. MAC Addresses

### Get MAC Address Using `ip link`

```bash
ip link show wlp2s0
```

Look for: `link/ether f8:59:71:a1:67:79`

### Get MAC Address Only

```bash
cat /sys/class/net/wlp2s0/address
```

**Output:**
```
f8:59:71:a1:67:79
```

### Why MAC Addresses Matter

- Unique hardware identifier
- Used for local network communication
- Required for DHCP
- Used in network policies and security
- Important for VM/container migration

---

## 6. Routing

### `ip route` - Show Routing Table

**Purpose:** Display the routing table.

```bash
ip route
```

**Sample Output:**
```
default via 192.168.1.1 dev wlp2s0 proto dhcp src 192.168.1.172 metric 600
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
172.19.0.0/16 dev br-feb0331de99f proto kernel scope link src 172.19.0.1
192.168.1.0/24 dev wlp2s0 proto kernel scope link src 192.168.1.172 metric 600
```

**Understanding Routes:**

```
default via 192.168.1.1 dev wlp2s0 proto dhcp src 192.168.1.172 metric 600
```

| Part | Meaning |
|------|---------|
| `default` | Default route (for unknown destinations) |
| `via 192.168.1.1` | Gateway IP address |
| `dev wlp2s0` | Interface to use |
| `proto dhcp` | Learned via DHCP |
| `src 192.168.1.172` | Source IP to use |
| `metric 600` | Priority (lower is better) |

**Route Types:**
- `default` - Default gateway
- `scope link` - Directly connected network
- `scope global` - Routable network
- `proto kernel` - Kernel-created route
- `proto static` - Manually added route

---

### `ip route get` - Test Route to Specific IP

**Purpose:** Show exactly how a specific IP would be reached.

```bash
ip route get 8.8.8.8
```

**Sample Output:**
```
8.8.8.8 via 192.168.1.1 dev wlp2s0 src 192.168.1.172 uid 1000
```

```bash
ip route get 172.19.0.5
```

**Sample Output:**
```
172.19.0.5 dev br-feb0331de99f src 172.19.0.1 uid 1000
```

**Why This Is Important:**
- No need to guess which interface will be used
- Shows exact path Linux would take
- Reveals the source IP that would be used
- More reliable than reading full routing table

---

### Find Default Gateway

```bash
ip route | grep default
```

**Output:**
```
default via 192.168.1.1 dev wlp2s0 proto dhcp src 192.168.1.172 metric 600
```

---

## 7. Bridges and Virtual Networking

### What Is a Linux Bridge?

A Linux bridge acts like a virtual Layer-2 switch, connecting multiple network interfaces (including virtual ones) like a physical switch.

```
┌────────────────────────────────────────────┐
│              Linux Bridge                  │
│              ┌──────────┐                  │
│              │ cni0     │                  │
│              │ 172.17.0.1│                  │
│              └────┬─────┘                  │
│         ┌────────┼────────┐                │
│         │        │        │                │
│      ┌──┴──┐ ┌───┴───┐ ┌─┴───┐            │
│      │veth │ │veth   │ │veth │            │
│      └──┬──┘ └───┬───┘ └─┬───┘            │
│         │        │        │                │
│      ┌──┴──┐ ┌───┴───┐ ┌─┴───┐            │
│      │Pod  │ │Pod   │ │Pod  │            │
│      └─────┘ └───────┘ └─────┘            │
└────────────────────────────────────────────┘
```

---

### `ip link show type bridge` - List Bridges

**Purpose:** Show only Linux bridges.

```bash
ip link show type bridge
```

**Sample Output:**
```
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
7: br-feb0331de99f: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
4: br-12f4cca121cf: <NO-CARRIER,BROADCAST,MULTICAST,UP> ... state DOWN
6: br-dae59a0b4dc3: <NO-CARRIER,BROADCAST,MULTICAST,UP> ... state DOWN
8: br-0555e251fc16: <NO-CARRIER,BROADCAST,MULTICAST,UP> ... state DOWN
```

**Understanding Bridge States:**
- `state UP` - Bridge is active
- `state DOWN` - Bridge is inactive
- `NO-CARRIER` - No active connections

---

### `bridge link` - Show Bridge Connections

**Purpose:** Show which interfaces are connected to each bridge.

```bash
bridge link
```

**Sample Output:**
```
9: veth2d30402@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-feb0331de99f state UP ...
10: veth38d4f79@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP ...
11: veth40976cb@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-feb0331de99f state UP ...
12: veth72f4630@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-feb0331de99f state UP ...
```

**Understanding:**
- `master br-feb0331de99f` - Connected to this bridge
- `state UP` - Interface is operational
- `veth` - Virtual Ethernet (container/pod interface)

---

### `ip addr show [bridge]` - Bridge IP Details

**Purpose:** Show IP configuration of a bridge.

```bash
ip addr show docker0
```

**Sample Output:**
```
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP ...
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
    inet6 fe80::1424:66ff:fef7:4e36/64 scope link
```

**What This Means:**
- Bridge has IP: `172.17.0.1`
- Network: `172.17.0.0/16`
- This IP acts as gateway for containers

---

## 8. Process Management

### `ps -ef` - All Processes

**Purpose:** Show all running processes with full command lines.

```bash
ps -ef
```

**Sample Output:**
```
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 10:30 ?        00:00:01 /sbin/init
root      2712     1  0 10:30 ?        00:00:05 /usr/bin/etcd
root      2762  2712  0 10:30 ?        00:00:03 kube-scheduler
root      1234  2712  0 10:30 ?        00:00:04 /usr/bin/dockerd
```

**Column Meaning:**
- `UID` - User who owns the process
- `PID` - Process ID
- `PPID` - Parent process ID
- `C` - CPU usage
- `STIME` - Start time
- `TTY` - Terminal
- `TIME` - CPU time used
- `CMD` - Command with arguments

---

### `ps aux` - Detailed Process View

**Purpose:** Show processes with resource usage.

```bash
ps aux
```

**Output Example:**
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      2712  0.5  2.1 123456 7890 ?        Ssl  10:30   0:05 /usr/bin/etcd
root      2762  0.1  1.2 98765 4567 ?        Ssl  10:30   0:03 kube-scheduler
```

**Column Meaning:**
- `USER` - Process owner
- `PID` - Process ID
- `%CPU` - CPU usage percentage
- `%MEM` - Memory usage percentage
- `VSZ` - Virtual memory size (KB)
- `RSS` - Resident memory size (KB)
- `STAT` - Process state
- `START` - Start time
- `TIME` - CPU time used
- `COMMAND` - Command

**Process States (STAT):**
- `R` - Running
- `S` - Sleeping
- `D` - Waiting for I/O
- `Z` - Zombie
- `T` - Stopped
- `s` - Session leader
- `l` - Multi-threaded
- `+` - Foreground process

---

### Filtering Processes

**Find specific service:**
```bash
ps -ef | grep etcd
```

**Output:**
```
root      2712     1  0 10:30 ?        00:00:05 /usr/bin/etcd
```

**Cleaner output (exclude grep):**
```bash
ps -ef | grep etcd | grep -v grep
```

**Find by process name only:**
```bash
ps -C etcd -o pid,cmd
```

**Find PID of a process:**
```bash
pgrep etcd
```
```
2712
```

---

### Process Startup Arguments

```bash
ps -ef | grep kube-scheduler | grep -v grep
```

**Sample Output:**
```
root 2762 1 0 10:30 ? 00:00:03 kube-scheduler \
    --authentication-kubeconfig=/etc/kubernetes/scheduler.conf \
    --authorization-kubeconfig=/etc/kubernetes/scheduler.conf \
    --bind-address=127.0.0.1 \
    --kubeconfig=/etc/kubernetes/scheduler.conf \
    --leader-elect=true
```

**What We Learn:**
- Binds to: `127.0.0.1:10259` (implied)
- Uses config file
- Leader election enabled

---

## 9. Socket Inspection (ss)

### What Is `ss`?

**`ss`** is the **Socket Statistics** tool. It replaced `netstat` because it's faster, more detailed, and shows more information.

**Why Use `ss` Over `netstat`:**
- Faster (reads directly from kernel)
- More information
- Better filtering options
- Handles large numbers of connections better

### `ss` Option Reference

| Option | Meaning | Use Case |
|--------|---------|----------|
| `-l` | LISTEN only | Find what's accepting connections |
| `-n` | Numeric (no DNS) | Faster, raw addresses |
| `-t` | TCP only | Focus on TCP |
| `-u` | UDP only | Focus on UDP |
| `-p` | Process info | Find owning process |
| `-a` | All sockets | Show everything |
| `-e` | Extended info | More details |
| `-o` | Timer info | Connection timers |
| `-4` | IPv4 only | Exclude IPv6 |
| `-6` | IPv6 only | Exclude IPv4 |
| `-A` | Query family | Options: `inet` (IPv4), `inet6`, `unix` |
| `-x` | UNIX sockets | Only UNIX domain sockets |
| `-s` | Summary | Summary of socket usage |

### `ss -lntp` - Listening TCP Ports

**Purpose:** Show all TCP sockets that are listening.

```bash
ss -lntp
```

**Sample Output:**
```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
LISTEN   0        4096     127.0.0.1:2379       0.0.0.0:*           users:(("etcd",pid=2712,fd=7))
LISTEN   0        4096     10.244.253.177:2380  0.0.0.0:*           users:(("etcd",pid=2712,fd=3))
LISTEN   0        128      127.0.0.1:10259      0.0.0.0:*           users:(("kube-scheduler",pid=2762,fd=3))
LISTEN   0        128      0.0.0.0:53           0.0.0.0:*           users:(("systemd-resolve",pid=456,fd=5))
LISTEN   0        128      0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=789,fd=3))
```

**Understanding Output:**

| Field | Meaning |
|-------|---------|
| `State` | Socket state (LISTEN, ESTAB, etc.) |
| `Recv-Q` | Data waiting to be read |
| `Send-Q` | Data waiting to be sent |
| `Local Address:Port` | Where it's listening |
| `Peer Address:Port` | Connected to (0.0.0.0:* for LISTEN) |
| `Process` | Process name, PID, and file descriptor |

**Socket States:**
- `LISTEN` - Waiting for connections
- `ESTAB` - Active connection
- `SYN-SENT` - Actively trying to connect
- `SYN-RECV` - Received SYN, sending SYN-ACK
- `FIN-WAIT-1/2` - Closing connection
- `TIME-WAIT` - Connection closed, waiting
- `CLOSE-WAIT` - Remote closed, waiting
- `CLOSED` - Socket closed

---

### `ss -ntp` - Active Connections

**Purpose:** Show all established TCP connections.

```bash
ss -ntp
```

**Sample Output:**
```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port    Process
ESTAB    0        0        127.0.0.1:2379       127.0.0.1:52341      users:(("kube-apiserver",pid=1234,fd=7))
ESTAB    0        0        192.168.1.172:52431  192.168.1.1:443      users:(("chrome",pid=7890,fd=23))
ESTAB    0        0        10.244.253.177:2380  10.244.127.244:2380  users:(("etcd",pid=2712,fd=8))
ESTAB    0        0        192.168.1.172:22     192.168.1.5:48923    users:(("sshd",pid=2345,fd=3))
```

**What This Shows:**
- Active connections between local and remote addresses
- Which process owns each connection
- Data queues (Recv-Q/Send-Q)

---

### `ss -lnup` - Listening UDP Ports

**Purpose:** Show all UDP sockets that are listening.

```bash
ss -lnup
```

**Sample Output:**
```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
UNCONN   0        0        0.0.0.0:53           0.0.0.0:*           users:(("systemd-resolve",pid=456,fd=5))
UNCONN   0        0        0.0.0.0:68           0.0.0.0:*           users:(("dhclient",pid=789,fd=4))
```

**UDP States:**
- `UNCONN` - Unconnected UDP socket (normal)
- `ESTAB` - Connected UDP socket (rare)

---

### Filtering `ss` Output

**Filter by service:**
```bash
ss -lntp | grep etcd
```

**Filter by port:**
```bash
ss -lntp | grep :2379
```

**Filter by process ID:**
```bash
ss -lntp | grep 2712
```

**Count connections to a port:**
```bash
ss -ntp | grep :2379 | wc -l
```

**Show only ESTABLISHED connections:**
```bash
ss -ntp | grep ESTAB
```

**Show connections to a specific IP:**
```bash
ss -ntp | grep "192.168.1.1"
```

---

### ETCD Ports - Important Distinction

| Port | Purpose | Who Connects |
|------|---------|--------------|
| **2379** | Client connections | kube-apiserver, etcdctl, Kubernetes components |
| **2380** | Peer connections | Other ETCD members |

**Checking Listening Ports:**
```bash
ss -lntp | grep etcd
```

**Checking Established Connections:**
```bash
ss -ntp | grep etcd | grep ESTAB
```

**Counting Connections:**
```bash
# Client connections
ss -ntp | grep etcd | grep :2379 | wc -l

# Peer connections
ss -ntp | grep etcd | grep :2380 | wc -l
```

---

### Understanding Recv-Q and Send-Q

```
Recv-Q   Send-Q
0        4096
```

| Field | Meaning | Problem Indicator |
|-------|---------|-------------------|
| `Recv-Q` | Bytes received but not read | If > 0 for long time, app not reading |
| `Send-Q` | Bytes sent but not acknowledged | If > 0 for long time, remote not receiving |

---

## 10. Advanced Networking Tools

### `ip neigh` - Neighbor/ARP Table

**Purpose:** Show ARP table (IP to MAC address mapping).

```bash
ip neigh
```

**Sample Output:**
```
192.168.1.1 dev wlp2s0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.5 dev wlp2s0 lladdr 11:22:33:44:55:66 STALE
192.168.1.20 dev wlp2s0  FAILED
```

**ARP States:**
- `REACHABLE` - Valid, verified reachability
- `STALE` - Valid but untested
- `DELAY` - Delay state before sending probes
- `PROBE` - Sending probes
- `FAILED` - Resolution failed
- `INCOMPLETE` - Waiting for response

**When to Use:**
- Local network connectivity issues
- MAC address verification
- ARP spoofing detection

---

### `ping` - Basic Connectivity

**Purpose:** Test ICMP connectivity.

```bash
ping 8.8.8.8
```

**Output:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=12.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=11.9 ms
```

**Testing a hostname:**
```bash
ping google.com
```

**Limited pings:**
```bash
ping -c 4 8.8.8.8
```

**Important:** `ping` uses ICMP and may be blocked. Success = ICMP works. Failure ≠ TCP service down.

---

### `nc` (Netcat) - TCP/UDP Testing

**Purpose:** Test TCP/UDP connectivity to a specific port.

```bash
nc -vz 192.168.1.1 22
```

**Output:**
```
Connection to 192.168.1.1 22 port [tcp/ssh] succeeded!
```

**Testing a closed port:**
```bash
nc -vz 192.168.1.1 9999
```
```
nc: connect to 192.168.1.1 port 9999 (tcp) failed: Connection refused
```

**Timeout with specific port:**
```bash
nc -vz -w 3 192.168.1.1 80
```

**Important Options:**
- `-v` - Verbose (show what's happening)
- `-z` - Zero I/O (just test, don't send data)
- `-u` - UDP mode
- `-w 3` - 3 second timeout

---

### `curl` - HTTP/HTTPS Testing

**Purpose:** Test HTTP/HTTPS services.

**Basic test:**
```bash
curl http://localhost:8080
```

**Kubernetes API test:**
```bash
curl -k https://127.0.0.1:6443
```
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "code": 401
}
```

**Get headers only:**
```bash
curl -I http://example.com
```

**Important Options:**
- `-k` - Ignore SSL certificate errors
- `-v` - Verbose output
- `-I` - Headers only- `-H "Header: value"` - Add header
- `-d '{"key":"value"}'` - Send POST data

---

### `tcpdump` - Packet Capture

**Purpose:** Capture and analyze network packets.

```bash
sudo tcpdump -i wlp2s0
```

**Capture specific port:**
```bash
sudo tcpdump -i wlp2s0 port 443
```

**Capture specific host:**
```bash
sudo tcpdump -i wlp2s0 host 192.168.1.1
```

**Capture and save to file:**
```bash
sudo tcpdump -i wlp2s0 -w capture.pcap
```

**Read saved capture:**
```bash
tcpdump -r capture.pcap
```

**Common Filters:**
- `port 80` - HTTP traffic
- `port 443` - HTTPS traffic
- `port 53` - DNS traffic
- `host 192.168.1.1` - Traffic to/from IP
- `src 192.168.1.1` - Source IP
- `dst 192.168.1.1` - Destination IP
- `tcp` - TCP packets
- `udp` - UDP packets
- `icmp` - ICMP packets (ping)

---

### `traceroute` / `tracepath` - Route Tracing

**Purpose:** Show the path packets take to reach a destination.

```bash
traceroute 8.8.8.8
```

**Output:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  2.123 ms  1.987 ms  2.456 ms
 2  10.0.0.1 (10.0.0.1)  5.678 ms  6.123 ms  5.890 ms
 3  172.16.0.1 (172.16.0.1)  8.901 ms  9.234 ms  8.678 ms
 ...
```

**Simpler version:**
```bash
tracepath 8.8.8.8
```

---

### `dig` / `nslookup` - DNS Resolution

**Purpose:** Query DNS servers.

```bash
dig google.com
```

**Output:**
```
;; ANSWER SECTION:
google.com.		300	IN	A	142.250.185.78
```

```bash
nslookup google.com
```

**Output:**
```
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	google.com
Address: 142.250.185.78
```

**Resolve from Kubernetes pod:**
```bash
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
```

---

### `getent` - System Database Query

**Purpose:** Query system databases including DNS.

```bash
getent hosts google.com
```
```
142.250.185.78  google.com
```

**Why Use `getent`:**
- Uses system resolver (same as applications)
- Can test both DNS and /etc/hosts
- Better for troubleshooting application DNS

---

### `resolvectl` - DNS Configuration

**Purpose:** Show DNS configuration on systemd systems.

```bash
resolvectl status
```

**Output:**
```
Global
       DNS Servers: 192.168.1.1
        DNSSEC: no
      DNS Domain: ~.
```

---

## 11. Container & Kubernetes Networking

### Pod/Container Networking Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Kubernetes Node                      │
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │   Pod A   │  │   Pod B   │  │   Pod C   │               │
│  │ 10.244.1.5│  │10.244.1.6 │  │10.244.1.7 │               │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘               │
│        │              │              │                      │
│    ┌───┴───┐      ┌───┴───┐      ┌───┴───┐                 │
│    │vethxxx│      │vethxxx│      │vethxxx│                 │
│    └───┬───┘      └───┬───┘      └───┬───┘                 │
│        │              │              │                      │
│        └──────────────┼──────────────┘                      │
│                       │                                     │
│                 ┌─────┴─────┐                               │
│                 │   cni0    │                               │
│                 │ 172.17.0.1│                               │
│                 └─────┬─────┘                               │
│                       │                                     │
│                  ┌────┴────┐                                │
│                  │ Flannel │                                │
│                  │ .1      │                                │
│                  └────┬────┘                                │
│                       │                                     │
│                 ┌─────┴─────┐                               │
│                 │   eth0    │                               │
│                 │10.244.1.1 │                               │
│                 └─────┬─────┘                               │
│                       │                                     │
└───────────────────────┼─────────────────────────────────────┘
                        │
             ┌──────────┴──────────┐
             │    Overlay Network   │
             │    (VXLAN, etc.)    │
             └──────────┬──────────┘
                        │
          ┌─────────────┴─────────────┐
          │       Other Nodes         │
          └───────────────────────────┘
```

---

### Kubernetes Commands for Networking

**Get Pod Information:**
```bash
kubectl get pods -o wide
```

**Output:**
```
NAME     READY   STATUS    RESTARTS   AGE   IP            NODE
pod-a    1/1     Running   0          2m    10.244.1.5    node01
pod-b    1/1     Running   0          2m    10.244.2.7    node02
```

**Get Node Information:**
```bash
kubectl get nodes -o wide
```

**Output:**
```
NAME           STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP
controlplane   Ready    master   10m   v1.28    10.244.253.177 <none>
node01         Ready    worker   8m    v1.28    10.244.127.244 <none>
```

**Get Services:**
```bash
kubectl get svc
```

**Output:**
```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    12m
webapp       ClusterIP   10.96.100.50    <none>        80/TCP     5m
```

**Get Endpoints:**
```bash
kubectl get endpoints
```

**Output:**
```
NAME         ENDPOINTS           AGE
kubernetes   10.244.253.177:6443 12m
webapp       10.244.1.5:80       5m
```

**Check Network Policies:**
```bash
kubectl get networkpolicy -A
```

**Describe Service:**
```bash
kubectl describe svc webapp
```

**Check CNI Components:**
```bash
kubectl get pods -n kube-system
```

---

### Testing Pod Connectivity

**From a pod to an IP:**
```bash
kubectl exec <pod> -- ping <destination-ip>
```

**From a pod to a service:**
```bash
kubectl exec <pod> -- curl <service-name>
```

**From a pod with netcat:**
```bash
kubectl exec <pod> -- nc -vz <destination-ip> <port>
```

**Check pod DNS:**
```bash
kubectl exec <pod> -- nslookup <service-name>
```

**Check pod resolv.conf:**
```bash
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

### CNI Inspection

**Find CNI Configuration:**
```bash
ls /etc/cni/net.d/
```
```
10-flannel.conflist
```

**Find CNI Binaries:**
```bash
ls /opt/cni/bin/
```
```
bridge  calico  calico-ipam  dhcp  flannel  host-local  ipvlan  macvlan  portmap  ptp  tuning
```

---

## 12. Complete Troubleshooting Workflows

### Scenario 1: "Service isn't accessible"

**Step 1: Is the service running?**
```bash
ps -ef | grep <service> | grep -v grep
```
**Step 2: Is it listening?**
```bash
ss -lntp | grep <service>
```
**Step 3: What IP and port?**
```bash
ss -lntp | grep <PID>
```
**Step 4: Are there connections?**
```bash
ss -ntp | grep <service>
```
**Step 5: Test locally:**
```bash
curl -k https://127.0.0.1:<port>
```
**Step 6: Check firewall:**
```bash
sudo iptables -L -n -v | grep <port>
```

---

### Scenario 2: "Container can't reach internet"

**Step 1: Check bridge:**
```bash
ip link show docker0
```

**Step 2: Check bridge IP:**
```bash
ip addr show docker0
```

**Step 3: Check container routes:**
```bash
docker exec <container> ip route
```

**Step 4: Check NAT rules:**
```bash
sudo iptables -t nat -L
```

**Step 5: Check default route:**
```bash
ip route
```

**Step 6: Test from container:**
```bash
docker exec <container> ping 8.8.8.8
docker exec <container> nslookup google.com
```

---

### Scenario 3: "Pod can't reach another pod"

**Step 1: Get pod IPs:**
```bash
kubectl get pods -o wide
```

**Step 2: Check same node or different:**
```bash
kubectl get pods -o wide | grep <pod-name>
```

**Step 3: On source node:**
```bash
ip route get <destination-pod-ip>
```

**Step 4: Check bridge:**
```bash
ip link show cni0
bridge link
```

**Step 5: Check CNI:**
```bash
kubectl get pods -n kube-system | grep -E 'flannel|calico|cilium'
```

**Step 6: Test connectivity:**
```bash
kubectl exec <source-pod> -- ping <destination-pod-ip>
```

**Step 7: Check NetworkPolicy:**
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <namespace>
```

---

### Scenario 4: "Pod can't resolve DNS"

**Step 1: Check CoreDNS:**
```bash
kubectl get pods -n kube-system | grep coredns
```

**Step 2: Check pod resolv.conf:**
```bash
kubectl exec <pod> -- cat /etc/resolv.conf
```

**Step 3: Test DNS from pod:**
```bash
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec <pod> -- nslookup google.com
```

**Step 4: Check kube-dns service:**
```bash
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns
```

**Step 5: Check CoreDNS logs:**
```bash
kubectl logs -n kube-system <coredns-pod>
```

---

### Scenario 5: "Pod can't reach service"

**Step 1: Check service exists:**
```bash
kubectl get svc <service-name>
```

**Step 2: Check endpoints:**
```bash
kubectl get endpoints <service-name>
```

**Step 3: Check pod labels:**
```bash
kubectl get pods --show-labels
kubectl describe svc <service-name>
```

**Step 4: Test DNS from pod:**
```bash
kubectl exec <pod> -- nslookup <service-name>
```

**Step 5: Test IP from pod:**
```bash
kubectl exec <pod> -- curl <cluster-ip>:<port>
```

**Step 6: Check kube-proxy:**
```bash
kubectl get pods -n kube-system | grep kube-proxy
```

---

### Scenario 6: "Node NotReady"

**Step 1: Check node status:**
```bash
kubectl get nodes
kubectl describe node <node>
```

**Step 2: On the node, check kubelet:**
```bash
systemctl status kubelet
journalctl -u kubelet -n 50
```

**Step 3: Check node network:**
```bash
ip addr
ip route
```

**Step 4: Check CNI:**
```bash
ls /etc/cni/net.d/
ls /opt/cni/bin/
```

**Step 5: Check container runtime:**
```bash
systemctl status docker
# or
systemctl status containerd
```

---

### Scenario 7: "ETCD connections failing"

**Step 1: Is ETCD running?**
```bash
ps -ef | grep etcd | grep -v grep
```

**Step 2: Is it listening?**
```bash
ss -lntp | grep etcd
```

**Step 3: Are there connections?**
```bash
ss -ntp | grep etcd
```

**Step 4: Count connections:**
```bash
# Client connections (2379)
ss -ntp | grep etcd | grep :2379 | wc -l

# Peer connections (2380)
ss -ntp | grep etcd | grep :2380 | wc -l
```

**Step 5: Check ETCD logs:**
```bash
journalctl -u etcd -n 50
```

**Step 6: Test from component:**
```bash
# Test kube-apiserver connection to etcd
kubectl get pods -n kube-system | grep apiserver
# Check apiserver logs
kubectl logs -n kube-system <apiserver-pod>
```

---

## 13. Quick Reference Cheat Sheet

### Linux Networking Commands

| Task | Command |
|------|---------|
| List interfaces | `ip link` |
| List interfaces with IPs | `ip addr` |
| Show specific interface | `ip addr show <interface>` |
| Show MAC address | `cat /sys/class/net/<interface>/address` |
| Interface state | `ip link show <interface>` |
| Routing table | `ip route` |
| Default gateway | `ip route \| grep default` |
| Route to specific IP | `ip route get <IP>` |
| Bridges | `ip link show type bridge` |
| Bridge members | `bridge link` |
| ARP table | `ip neigh` |
| Running processes | `ps -ef` |
| Specific process | `ps -ef \| grep <name> \| grep -v grep` |
| Listening TCP | `ss -lnt` |
| Listening TCP with processes | `ss -lntp` |
| Listening UDP | `ss -lnup` |
| Listening UDP with processes | `ss -lnup` |
| Established TCP | `ss -ntp` |
| Filter by port | `ss -lntp \| grep :<port>` |
| Filter by process | `ss -lntp \| grep <process>` |
| Test ICMP | `ping <IP>` |
| Test TCP | `nc -vz <IP> <port>` |
| Test HTTP | `curl http://<URL>` |
| DNS lookup | `dig <name>` |
| DNS test | `nslookup <name>` |
| System DNS | `getent hosts <name>` |
| DNS config | `resolvectl status` |
| Packet capture | `tcpdump -i <interface>` |
| Route trace | `traceroute <IP>` |
| Interface stats | `ip -s link show <interface>` |
| Link speed | `ethtool <interface>` |
| Firewall | `iptables -L -n -v` |

### Kubernetes Networking Commands

| Task | Command |
|------|---------|
| Node IPs | `kubectl get nodes -o wide` |
| Pod IPs | `kubectl get pods -o wide` |
| Services | `kubectl get svc` |
| Service endpoints | `kubectl get endpoints` |
| Network policies | `kubectl get networkpolicy -A` |
| CNI components | `kubectl get pods -n kube-system` |
| Node details | `kubectl describe node <node>` |
| Service details | `kubectl describe svc <service>` |
| Pod logs | `kubectl logs <pod>` |
| Debug pod | `kubectl run debug --image=busybox -it --rm -- sh` |

---

## 14. Common Ports Reference

### Kubernetes Ports

| Port | Component | Purpose |
|------|-----------|---------|
| 6443 | kube-apiserver | API Server |
| 10250 | kubelet | Node agent |
| 10257 | kube-controller-manager | Controller manager |
| 10259 | kube-scheduler | Scheduler |
| 2379 | etcd | Client connections |
| 2380 | etcd | Peer connections |
| 53 | CoreDNS | DNS resolution |
| 443 | Ingress/Service | HTTPS |
| 80 | Ingress/Service | HTTP |
| 9100 | node-exporter | Metrics |
| 9090 | prometheus | Metrics server |
| 30000-32767 | NodePort | NodePort services |

### Common System Ports

| Port | Service |
|------|---------|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 8080 | HTTP alternate |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |

---

## 15. Final Mental Model

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│    "What question am I asking, and which command            │
│     answers that question?"                                 │
│                                                             │
│  What interfaces?         → ip link                        │
│  What IPs?                → ip addr                        │
│  What's the MAC?          → ip link / sys/class/net/       │
│  What's the route?        → ip route                       │
│  Which interface for IP?  → ip route get <IP>             │
│  What's connected?        → bridge link                    │
│  What's running?          → ps -ef                         │
│  What's listening?        → ss -lntp                       │
│  Who's connected?         → ss -ntp                        │
│  Can we reach it?         → ping <IP>                      │
│  Is the port open?        → nc -vz <IP> <port>            │
│  Is HTTP working?         → curl <URL>                     │
│  DNS working?             → dig / nslookup                 │
│  What are packets doing?  → tcpdump -i <interface>        │
│  What are pods?           → kubectl get pods -o wide      │
│  What are services?       → kubectl get svc               │
│  Are endpoints there?     → kubectl get endpoints         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 16. Golden Rules

1. **Never assume interface names** - Always inspect the actual system
2. **Start at the bottom** - Interface → IP → Route → Connectivity → Process → Service
3. **Check the obvious first** - Is it running? Is it listening? Can I reach it?
4. **Understand LISTEN vs ESTAB** - Listening = waiting, Established = connected
5. **Use `ip route get`** - Don't guess which interface, ask Linux
6. **Test at the right layer** - `ping` for ICMP, `nc` for TCP, `curl` for HTTP
7. **Check both ends** - Server listening AND client connecting
8. **DNS matters** - Check both DNS resolution and connectivity
9. **Kubernetes is layers** - Pod → veth → bridge → CNI → node → network
10. **Logs are your friend** - `kubectl logs`, `journalctl`, `tail -f /var/log/...`

---
