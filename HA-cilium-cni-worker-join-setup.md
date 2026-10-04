# Kubernetes HA — Cilium eBPF / kube-proxy Replacement, Workers Join

**Complete Runbook — Installation, Troubleshooting, and Migration**

---

## Reading This Document

This document intentionally shows:

- ✅ **CORRECT** commands
- ❌ **WRONG** commands we ran
- 🔧 **FIXES** we applied
- 📌 **INCOMPLETE** places where we stopped and had to investigate further

When something is marked **WRONG** or **INCOMPLETE**, the correct version follows immediately after.

This mirrors the actual troubleshooting journey, which is the most valuable part for future projects and interviews.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Prerequisites](#2-prerequisites)
3. [Final Architecture](#3-final-architecture)
4. [Kubernetes Network Configuration](#4-kubernetes-network-configuration)
5. [Why Cilium?](#5-why-cilium)
6. [Cilium Networking Design](#6-cilium-networking-design)
7. [Geneve Networking](#7-geneve-networking)
8. [Security Group Requirements](#8-security-group-requirements)
9. [Cilium Version Selection](#9-cilium-version-selection)
10. [Initial Cilium Installation](#10-initial-cilium-installation)
11. [First Problem — Cilium Could Not Reach the Kubernetes API](#11-first-problem--cilium-could-not-reach-the-kubernetes-api)
12. [kube-proxy Was Still Running](#12-kube-proxy-was-still-running)
13. [Cilium API Endpoint Configuration](#13-cilium-api-endpoint-configuration)
14. [Second Problem — TLS Certificate Verification Failure](#14-second-problem--tls-certificate-verification-failure)
15. [How We Proved the Certificate Problem](#15-how-we-proved-the-certificate-problem)
16. [Why Did the Certificate Contain the HAProxy DNS Name?](#16-why-did-the-certificate-contain-the-haproxy-dns-name)
17. [Inspecting the API Server Certificate Directly](#17-inspecting-the-api-server-certificate-directly)
18. [Root Cause](#18-root-cause)
19. [Fixing the TLS Problem](#19-fixing-the-tls-problem)
20. [Important TLS Lesson](#20-important-tls-lesson)
21. [Security Group Troubleshooting](#21-security-group-troubleshooting)
22. [Verifying the Cilium Configuration](#22-verifying-the-cilium-configuration)
23. [Final Cilium Health Verification](#23-final-cilium-health-verification)
24. [Final Cilium Node Verification](#24-final-cilium-node-verification)
25. [Cilium Components](#25-cilium-components)
26. [Migration Strategy](#26-migration-strategy)
27. [Before/After ClusterIP Proof](#27-beforeafter-clusterip-proof)
28. [Removing kube-proxy](#28-removing-kube-proxy)
29. [Cleaning kube-proxy iptables Rules](#29-cleaning-kube-proxy-iptables-rules)
30. [Final kube-proxy Verification](#30-final-kube-proxy-verification)
31. [Final Architecture](#31-final-architecture)
32. [Joining Worker Nodes](#32-joining-worker-nodes)
33. [Worker Join Troubleshooting — The HAProxy Security Group Issue](#33-worker-join-troubleshooting--the-haproxy-security-group-issue)
34. [Mistakes Made During the Build](#34-mistakes-made-during-the-build)
35. [Rollback Procedure](#35-rollback-procedure)
36. [Troubleshooting Commands Worth Keeping](#36-troubleshooting-commands-worth-keeping)
37. [Final Verification Checklist](#37-final-verification-checklist)
38. [Important Interview Concepts](#38-important-interview-concepts)
39. [Key Lessons From This Lab](#39-key-lessons-from-this-lab)
40. [Final State](#40-final-state)

---

## 1. Purpose

This document records the installation, configuration, troubleshooting, and migration of Cilium as the Kubernetes networking and service dataplane for our self-managed HA Kubernetes cluster.

The objective was to build a Kubernetes cluster where:

- Cilium provides pod networking.
- Cilium uses eBPF for Kubernetes Service handling.
- Cilium replaces kube-proxy.
- Geneve is used as the Cilium tunnel protocol.
- Cilium uses cluster-pool IPAM.
- The Kubernetes API is accessed through the HAProxy HA endpoint.
- kube-proxy is completely removed.
- The cluster remains functional after kube-proxy removal.
- Two worker nodes are joined and labeled.

This document intentionally records the mistakes and troubleshooting process because those are the most valuable parts for future projects and interviews.

---

## 2. Prerequisites

Before starting, ensure the following are in place:

| Requirement | Verification |
|-------------|--------------|
| Kubernetes cluster initialized with kubeadm | `kubectl get nodes` |
| All control-plane nodes reachable | `kubectl get nodes -o wide` |
| `kubectl` configured on bastion | `kubectl cluster-info` |
| Helm 3.x installed on bastion | `helm version` |
| HAProxy endpoint reachable from all nodes | `nc -zv haproxy.faekcorp.lab 6443` |
| DNS resolves `haproxy.faekcorp.lab` | `dig haproxy.faekcorp.lab` |
| Security Groups permit UDP 6081, TCP 4240, TCP 6443 | See Section 8 |
| No other CNI installed | `kubectl get pods -A \| grep -E 'calico\|flannel\|weave'` |

**Administration host:** Bastion (kubectl + helm)

**Cluster nodes:**

```
cp01.faekcorp.lab    10.70.21.6
cp02.faekcorp.lab    10.70.31.209
cp03.faekcorp.lab    10.70.41.250
worker1.faekcorp.lab 10.70.61.85
worker2.faekcorp.lab 10.70.11.67
```

---

## 3. Final Architecture

### AWS topology

```text
                         ┌──────────────────┐
                         │     Bastion      │
                         │                  │
                         │ kubectl / helm   │
                         └────────┬─────────┘
                                  │
                                  │ TCP 6443
                                  ▼
                       ┌──────────────────────┐
                       │ HAProxy + BIND9      │
                       │                      │
                       │ 10.70.11.192         │
                       │ haproxy.faekcorp.lab │
                       └──────────┬───────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             ▼             ▼
              ┌──────────┐  ┌──────────┐  ┌──────────┐
              │   cp01   │  │   cp02   │  │   cp03   │
              │10.70.21.6│  │10.70.31.209│ │10.70.41.250│
              └──────────┘  └──────────┘  └──────────┘
                    │             │             │
                    └─────────────┼─────────────┘
                                  │
                          Cilium 1.20.1
                                  │
              ┌───────────────────┴───────────────────┐
              │                                       │
         eBPF datapath                          Geneve tunnel
         Service LB                             UDP 6081
              │                                       │
              └───────────────────┬───────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                 worker1                     worker2
              10.70.61.85                 10.70.11.67
                    │                           │
                    └─────────────┬─────────────┘
                                  │
                          Pod network
                         10.244.0.0/16
```

The Kubernetes API endpoint is:

```
haproxy.faekcorp.lab:6443
```

HAProxy distributes API traffic to:

```
cp01:6443
cp02:6443
cp03:6443
```

---

## 4. Kubernetes Network Configuration

The cluster was initialized with:

```yaml
kubernetesVersion: v1.35.8

networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12

proxy: {}
scheduler: {}
```

Important values:

| Component           | Value                       |
| ------------------- | --------------------------- |
| Kubernetes          | v1.35.8                     |
| Pod CIDR            | `10.244.0.0/16`             |
| Service CIDR        | `10.96.0.0/12`              |
| DNS domain          | `cluster.local`             |
| API HA endpoint     | `haproxy.faekcorp.lab:6443` |
| HAProxy IP          | `10.70.11.192`              |
| CNI                 | Cilium 1.20.1               |
| IPAM                | Cluster Pool                |
| Routing             | Tunnel                      |
| Tunnel              | Geneve                      |
| Service replacement | eBPF                        |
| kube-proxy          | Removed                     |

---

## 5. Why Cilium?

The goal was not simply to install another CNI.

We wanted Cilium to provide:

```
Pod networking
      +
Network policy
      +
eBPF datapath
      +
Kubernetes Service handling
      +
kube-proxy replacement
```

Traditional Kubernetes networking commonly looks like:

```
Pod
 |
 v
Service ClusterIP
 |
 v
kube-proxy
 |
 v
iptables/IPVS
 |
 v
Backend Pod
```

With Cilium kube-proxy replacement:

```
Pod
 |
 v
Service ClusterIP
 |
 v
Cilium eBPF
 |
 v
Backend Pod
```

The objective was therefore to eliminate kube-proxy and allow Cilium's eBPF datapath to handle Kubernetes Services.

---

## 6. Cilium Networking Design

### 6.1 IPAM

We selected:

```
cluster-pool
```

with:

```
10.244.0.0/16
```

Cilium divides this pool into smaller per-node blocks.

For example, our nodes received:

```
cp01 → 10.244.0.0/24
cp02 → 10.244.1.0/24
cp03 → 10.244.2.0/24
```

We verified this with:

```bash
kubectl get ciliumnodes
```

Result:

```
NAME                CILIUMINTERNALIP   INTERNALIP
cp01.faekcorp.lab   10.244.0.140       10.70.21.6
cp02.faekcorp.lab   10.244.1.187       10.70.31.209
cp03.faekcorp.lab   10.244.2.193       10.70.41.250
```

The important point is that the `CILIUMINTERNALIP` is from the Kubernetes pod network while `INTERNALIP` is the EC2/node network.

---

## 7. Geneve Networking

We selected:

```
routingMode=tunnel
tunnelProtocol=geneve
```

Geneve encapsulates pod traffic between nodes.

Conceptually:

```
Pod A
10.244.x.x
   |
   | Cilium
   v
Geneve encapsulation
   |
   | UDP 6081
   v
AWS VPC
   |
   v
Destination node
   |
   v
Pod B
10.244.y.y
```

Therefore the Security Groups need to permit:

```
UDP 6081
```

between Kubernetes nodes.

---

## 8. Security Group Requirements

The following Security Group rules are required for Cilium to function correctly:

| Source SG | Destination SG | Protocol | Port | Purpose |
|-----------|---------------|----------|------|---------|
| cp-sg | cp-sg | UDP | 6081 | Geneve tunnel |
| cp-sg | worker-sg | UDP | 6081 | Geneve tunnel |
| worker-sg | cp-sg | UDP | 6081 | Geneve tunnel |
| worker-sg | worker-sg | UDP | 6081 | Geneve tunnel |
| cp-sg | cp-sg | TCP | 4240 | Cilium health checks |
| cp-sg | worker-sg | TCP | 4240 | Cilium health checks |
| worker-sg | cp-sg | TCP | 4240 | Cilium health checks |
| worker-sg | worker-sg | TCP | 4240 | Cilium health checks |
| Any | cp-sg | TCP | 6443 | Kubernetes API |
| cp-sg | cp-sg | TCP | 2379-2380 | etcd |
| cp-sg | cp-sg | TCP | 10250 | kubelet API |
| cp-sg | cp-sg | TCP | 10257 | kube-controller-manager |
| cp-sg | cp-sg | TCP | 10259 | kube-scheduler |
| **worker-sg** | **lb-dns-sg** | **TCP** | **6443** | **Worker → HAProxy API access** |

### ⚠️ Important missing rule we discovered

We initially omitted:

```
worker-sg → lb-dns-sg : TCP 6443
```

This caused `kubeadm join` to hang at `[preflight] Running pre-flight checks` on worker nodes.

**Troubleshooting lesson:** Do not assume a Security Group is the problem simply because a pod is not healthy. But conversely — if `kubeadm join` hangs at preflight, **check the Security Group path from worker to HAProxy first.**

---

## 9. Cilium Version Selection

We selected Cilium **1.20.1** based on the following criteria:

| Factor | Consideration |
|--------|---------------|
| Kubernetes compatibility | Cilium 1.20.x supports Kubernetes 1.30–1.35 |
| Stability | 1.20.1 is a patch release with bug fixes |
| Feature set | Supports kube-proxy replacement, Geneve, cluster-pool IPAM |
| Hubble | Included and stable in 1.20.x |
| Latest vs stable | 1.20.2 was available but 1.20.1 was chosen for this lab |

**Verification:**

```bash
helm search repo cilium/cilium --versions
```

Output (truncated):

```
NAME            CHART VERSION   APP VERSION   DESCRIPTION
cilium/cilium   1.20.2          1.20.2        eBPF-based Networking, Security, and Observability
cilium/cilium   1.20.1          1.20.1        eBPF-based Networking, Security, and Observability
cilium/cilium   1.20.0          1.20.0        eBPF-based Networking, Security, and Observability
cilium/cilium   1.19.8          1.19.8        eBPF-based Networking, Security, and Observability
...
```

**Note:** Always check the [Cilium Kubernetes compatibility matrix](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/) before selecting a version.

---

## 10. Initial Cilium Installation

### 10.1 Add the Cilium Helm repository

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

### 10.2 Create the values file

Rather than using many `--set` flags, we use a `cilium-values.yaml` file for reproducibility:

```yaml
# values/cilium-values.yaml
operator:
  replicas: 1

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.244.0.0/16

kubeProxyReplacement: true

routingMode: tunnel
tunnelProtocol: geneve

k8sServiceHost: haproxy.faekcorp.lab
k8sServicePort: 6443
```

### 10.3 Install Cilium

```bash
helm upgrade --install cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --create-namespace \
  --values values/cilium-values.yaml
```

### 10.4 Equivalent `--set` form (for reference)

If you prefer flags over a values file, this is the equivalent:

```bash
helm upgrade --install cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --create-namespace \
  --set operator.replicas=1 \
  --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList[0]=10.244.0.0/16 \
  --set kubeProxyReplacement=true \
  --set routingMode=tunnel \
  --set tunnelProtocol=geneve
```

### 10.5 What this configures

| Setting | Value |
|---------|-------|
| IPAM | cluster-pool |
| Pod pool | 10.244.0.0/16 |
| kube-proxy replacement | true |
| Routing | tunnel |
| Tunnel | Geneve |
| Operator replicas | 1 |

---

## 11. First Problem — Cilium Could Not Reach the Kubernetes API

After installation, Cilium agents were not becoming healthy.

The important error was:

```
dial tcp 10.96.0.1:443: i/o timeout
```

The address:

```
10.96.0.1:443
```

is the Kubernetes Service IP for the API server.

### ⚠️ INCOMPLETE at this point

At this stage we did not yet know whether the problem was:

- The Service (wrong endpoints)
- Security Groups
- TLS
- Cilium misconfiguration

We needed more evidence.

We first verified the Kubernetes Service:

```bash
kubectl describe svc kubernetes
```

Result:

```
Name:         kubernetes
Namespace:    default
Type:         ClusterIP
IP:           10.96.0.1
Port:         https 443/TCP
TargetPort:   6443
Endpoints:
  10.70.21.6:6443
  10.70.31.209:6443
  10.70.41.250:6443
```

We also checked:

```bash
kubectl get endpoints
```

Result:

```
kubernetes
10.70.21.6:6443
10.70.31.209:6443
10.70.41.250:6443
```

This proved something important:

> The Kubernetes Service itself was correctly configured and the API endpoints existed.

The problem was therefore **not** that the API servers were missing.

---

## 12. kube-proxy Was Still Running

We then discovered that kube-proxy was still installed:

```bash
kubectl get ds -A
```

Result:

```
cilium        3   3   3
cilium-envoy  3   3   3
kube-proxy    3   3   3
```

And:

```bash
kubectl -n kube-system get pods
```

showed:

```
kube-proxy-fz5fg
kube-proxy-l4jnx
kube-proxy-ssm4q
```

This was an important configuration mistake.

### Why did kube-proxy exist?

The cluster had been initialized without:

```bash
--skip-phases=addon/kube-proxy
```

Therefore kubeadm installed kube-proxy normally.

The following:

```yaml
proxy: {}
```

in the kubeadm configuration did **not** mean:

> Do not install kube-proxy.

### ❌ WRONG assumption

> "Setting `proxy: {}` in the kubeadm config disables kube-proxy installation."

This is **incorrect**. kube-proxy is installed by default by kubeadm unless you explicitly skip it.

### ✅ CORRECT approach for a fresh cluster

For a brand-new kubeadm cluster intended to be kube-proxy-free:

**Option A — CLI flag:**

```bash
kubeadm init --skip-phases=addon/kube-proxy
```

**Option B — Config file:**

```yaml
# kubeadm-config.yaml
kind: ClusterConfiguration
...
---
kind: InitConfiguration
...
skipPhases:
  - addon/kube-proxy
```

Then:

```bash
kubeadm init --config kubeadm-config.yaml
```

### Our approach

Because this cluster was already running, we chose to perform a controlled migration instead of rebuilding the cluster. This is documented in Sections 26–30.

---

## 13. Cilium API Endpoint Configuration

For kube-proxy replacement, Cilium needs a reliable Kubernetes API endpoint.

We configured:

```
k8sServiceHost
k8sServicePort
```

### ❌ WRONG initial value

We initially used the HAProxy IP:

```
10.70.11.192:6443
```

We applied:

```bash
helm upgrade cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --reuse-values \
  --set k8sServiceHost=10.70.11.192 \
  --set k8sServicePort=6443
```

We verified Helm values:

```bash
helm get values cilium -n kube-system
```

The important values were:

```yaml
k8sServiceHost: 10.70.11.192
k8sServicePort: 6443
kubeProxyReplacement: true
```

**This looked correct, but it caused the TLS problem in Section 14.**

---

## 14. Second Problem — TLS Certificate Verification Failure

This was the most important troubleshooting discovery.

Cilium logs showed:

```
Establishing connection to apiserver
ipAddr=https://10.70.11.192:6443
```

followed by:

```
tls: failed to verify certificate:
x509: certificate is valid for 10.96.0.1, 10.70.31.209,
not 10.70.11.192
```

This error was extremely valuable.

It told us:

```
TCP connection succeeded
        |
        v
TLS handshake started
        |
        v
Certificate received
        |
        v
Certificate identity verification failed
```

Therefore this was **not primarily a Security Group problem**.

If Security Groups were blocking the connection, we would expect a network timeout or connection failure.

Instead, we successfully reached the API server but TLS rejected its certificate.

### 📌 Key observation

The error message is precise:

```
certificate is valid for 10.96.0.1, 10.70.31.209,
not 10.70.11.192
```

It tells us exactly which SANs the certificate **does** contain, and which one we requested.

---

## 15. How We Proved the Certificate Problem

We inspected the certificate presented through HAProxy:

```bash
openssl s_client \
  -connect 10.70.11.192:6443 \
  -showcerts </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -ext subjectAltName
```

Output:

```
subject=CN = kube-apiserver
issuer=CN = kubernetes

X509v3 Subject Alternative Name:
    DNS:cp02.faekcorp.lab,
    DNS:haproxy.faekcorp.lab,
    DNS:kubernetes,
    DNS:kubernetes.default,
    DNS:kubernetes.default.svc,
    DNS:kubernetes.default.svc.cluster.local,
    IP Address:10.96.0.1,
    IP Address:10.70.31.209
```

The critical observation:

```
DNS:haproxy.faekcorp.lab
```

was present.

But:

```
IP Address:10.70.11.192
```

was NOT present.

Therefore:

```
https://10.70.11.192:6443
```

could not pass certificate hostname/IP verification.

But:

```
https://haproxy.faekcorp.lab:6443
```

could.

---

## 16. Why Did the Certificate Contain the HAProxy DNS Name?

We checked the kubeadm configuration:

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml
```

We found:

```yaml
controlPlaneEndpoint: haproxy.faekcorp.lab:6443
```

This is exactly what we wanted.

The Kubernetes API certificate was therefore generated with the HA endpoint DNS name:

```
haproxy.faekcorp.lab
```

The mistake was not in the kubeadm configuration.

The mistake was that we later configured Cilium to access the API using:

```
10.70.11.192
```

instead of the DNS name already represented in the certificate.

---

## 17. Inspecting the API Server Certificate Directly

We also checked the certificate directly on cp02:

```bash
sudo openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout \
  -subject \
  -issuer \
  -ext subjectAltName
```

The result again showed:

```
DNS:haproxy.faekcorp.lab
```

but no:

```
IP Address:10.70.11.192
```

This confirmed that HAProxy was not modifying the certificate.

HAProxy was simply forwarding the Kubernetes API TLS connection.

---

## 18. Root Cause

The complete chain was:

```
Cilium
   |
   | configured API endpoint
   |
   v
https://10.70.11.192:6443
   |
   v
HAProxy
   |
   v
Kubernetes API server
   |
   v
Certificate SANs
   |
   +---- haproxy.faekcorp.lab      YES
   |
   +---- 10.70.11.192             NO
```

TLS therefore rejected the connection.

The correct architecture was:

```
Cilium
   |
   | https://haproxy.faekcorp.lab:6443
   v
DNS
   |
   v
10.70.11.192
   |
   v
HAProxy
   |
   +---- cp01:6443
   +---- cp02:6443
   +---- cp03:6443
```

---

## 19. Fixing the TLS Problem

Instead of regenerating certificates, we used the existing valid DNS SAN.

### ✅ CORRECT fix

We changed Cilium to:

```bash
helm upgrade cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --reuse-values \
  --set k8sServiceHost=haproxy.faekcorp.lab \
  --set k8sServicePort=6443
```

This was the clean fix.

We did NOT need to regenerate:

```
apiserver.crt
```

because the certificate already contained:

```
DNS:haproxy.faekcorp.lab
```

### Why we did not regenerate the certificate

Regenerating `apiserver.crt` would have required:

- Updating all kubeconfig files
- Restarting all control-plane components
- Potentially breaking existing clients

The DNS SAN was **already present** — we simply were not using it.

---

## 20. Important TLS Lesson

When Kubernetes components communicate with an HTTPS endpoint, the hostname used by the client must match a SAN in the server certificate.

For example:

```
Certificate:
DNS:haproxy.faekcorp.lab
```

Valid:

```
https://haproxy.faekcorp.lab:6443
```

Not valid:

```
https://10.70.11.192:6443
```

unless the certificate also contains:

```
IP Address:10.70.11.192
```

### General troubleshooting rule

When you see:

```
x509:
certificate is valid for X,
not Y
```

think:

```
TLS identity mismatch
```

not:

```
Security Group problem
```

### Error signature table

Each layer has a distinct signature. Learn the signatures, and the layer identifies itself:

| Error | Layer | Common Cause |
|-------|-------|--------------|
| `DNS resolution failed` | DNS | Wrong name, no record, resolver unreachable |
| `no route to host` | L3 routing | Missing route, wrong subnet |
| `i/o timeout` | L3/L4 firewall | Security Group, iptables, NACL dropping packets |
| `connection refused` | L4 | Nothing listening, wrong port, process down |
| `connection reset` | L4 | TCP RST, often SG stateful rules or proxy |
| `x509: certificate is valid for X, not Y` | TLS | SAN mismatch, wrong hostname/IP used |
| `tls: handshake failure` | TLS | Protocol/cipher mismatch |
| `401 Unauthorized` | HTTP/API | Bad token, expired cert |
| `403 Forbidden` | HTTP/API | RBAC, admission webhook |
| `kubeadm join` stuck at preflight | Network/SG | Worker cannot reach HAProxy on 6443 |

---

## 21. Security Group Troubleshooting

The cluster also required Geneve networking.

Cilium Geneve uses:

```
UDP 6081
```

The node Security Groups therefore allowed UDP 6081 between the control-plane and worker node groups.

Relevant Cilium health traffic also included:

```
TCP 4240
```

and Kubernetes control-plane traffic included:

```
TCP 6443
```

The important troubleshooting lesson was:

> Do not assume a Security Group is the problem simply because a pod is not healthy.

We used the actual error to determine the layer that was failing.

The TLS error:

```
x509: certificate is valid for ...
not ...
```

proved that packets were reaching the API server far enough to perform TLS negotiation.

Therefore we investigated certificates instead of blindly changing Security Groups.

**However**, a Security Group issue was later the cause of the worker join problem (see Section 33).

---

## 22. Verifying the Cilium Configuration

We checked:

```bash
kubectl get cm cilium-config -n kube-system -o yaml
```

Important values included:

```
cluster-pool-ipv4-cidr: 10.244.0.0/16
cluster-pool-ipv4-mask-size: "24"
ipam: cluster-pool
kube-proxy-replacement: "true"
routing-mode: tunnel
tunnel-protocol: geneve
```

We also used:

```bash
helm get values cilium -n kube-system
```

which showed:

```yaml
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
    - 10.244.0.0/16

k8sServiceHost: haproxy.faekcorp.lab
k8sServicePort: 6443

kubeProxyReplacement: true

operator:
  replicas: 1

routingMode: tunnel
tunnelProtocol: geneve
```

**Note:** `k8sServiceHost` and `k8sServicePort` are Helm configuration values and do not necessarily need to appear as literal keys in the Cilium ConfigMap. `helm get values` is the appropriate way to verify the Helm release values.

---

## 23. Final Cilium Health Verification

Once the DNS endpoint was used, Cilium became healthy.

We checked:

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- \
  cilium-dbg status
```

Important output:

```
KVStore:                 Disabled
Kubernetes:              Ok         1.35 (v1.35.8)
KubeProxyReplacement:    True
Cilium:                  Ok          1.20.1
Cilium health daemon:    Ok
IPAM:                    IPv4: 4/254 allocated from 10.244.2.0/24
Routing:                 Network: Tunnel [geneve]
Controller Status:       26/26 healthy
Proxy Status:            OK
Hubble:                  Ok
Cluster health:          3/3 reachable
Modules Health:          Stopped(0) Degraded(0) OK(81)
```

The most important lines were:

```
Kubernetes:              Ok
```

This means Cilium can communicate with the Kubernetes API.

```
KubeProxyReplacement:    True
```

This means Cilium's kube-proxy replacement is enabled.

```
Routing:                 Network: Tunnel [geneve]
```

This confirms Geneve tunneling.

```
Cluster health:          3/3 reachable
```

This confirms Cilium connectivity between all three control-plane nodes.

---

## 24. Final Cilium Node Verification

We ran:

```bash
kubectl get ciliumnodes
```

Result:

```
NAME                CILIUMINTERNALIP   INTERNALIP
cp01.faekcorp.lab   10.244.0.140       10.70.21.6
cp02.faekcorp.lab   10.244.1.187       10.70.31.209
cp03.faekcorp.lab   10.244.2.193       10.70.41.250
```

This confirmed Cilium had successfully registered all three nodes.

---

## 25. Cilium Components

We verified:

```bash
kubectl get ds -A
```

Before migration:

```
cilium        3   3   3
cilium-envoy  3   3   3
kube-proxy    3   3   3
```

Cilium itself was healthy before removing kube-proxy.

We also verified:

```bash
kubectl get deploy -A
```

Result:

```
kube-system   cilium-operator   1/1
kube-system   coredns            2/2
```

---

## 26. Migration Strategy

We deliberately did not immediately delete kube-proxy.

The migration sequence was:

```
1. Install Cilium
       |
       v
2. Enable kube-proxy replacement
       |
       v
3. Fix API connectivity
       |
       v
4. Fix TLS certificate mismatch
       |
       v
5. Verify Cilium health
       |
       v
6. Verify cluster health 3/3
       |
       v
7. Confirm KubeProxyReplacement=True
       |
       v
8. Prove ClusterIP works BEFORE removing kube-proxy
       |
       v
9. Remove kube-proxy
       |
       v
10. Remove kube-proxy ConfigMap
       |
       v
11. Clean old KUBE iptables rules
       |
       v
12. Verify kube-proxy is gone
       |
       v
13. Prove ClusterIP works AFTER removing kube-proxy
       |
       v
14. Join worker nodes
       |
       v
15. Label workers
       |
       v
16. Verify Cilium on all 5 nodes
       |
       v
17. Final verification
```

This is safer than deleting kube-proxy while Cilium is still unhealthy.

---

## 27. Before/After ClusterIP Proof

To prove that Cilium handles Kubernetes Services, we create a test workload and verify ClusterIP connectivity both before and after removing kube-proxy.

### 27.1 Create the test namespace and workload

```bash
kubectl create namespace svc-test
```

Create the manifest:

```yaml
# manifests/svc-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: svc-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: svc-test
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f manifests/svc-test.yaml
```

### 27.2 Verify the workload

```bash
kubectl -n svc-test get pods -o wide
kubectl -n svc-test get svc nginx
kubectl -n svc-test get endpoints nginx
```

Expected:

```
NAME    TYPE        CLUSTER-IP     PORT(S)
nginx   ClusterIP   10.x.x.x       80/TCP
```

And two Pod IPs behind the Service.

### 27.3 Test BEFORE removing kube-proxy

```bash
kubectl -n svc-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://nginx.svc-test.svc.cluster.local
```

Expected output: nginx welcome page HTML.

### 27.4 Remove kube-proxy (Sections 28–29)

### 27.5 Test AFTER removing kube-proxy

Run the exact same command:

```bash
kubectl -n svc-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://nginx.svc-test.svc.cluster.local
```

Expected output: the same nginx welcome page HTML.

**This proves** that Cilium's eBPF datapath is handling ClusterIP Services, not kube-proxy.

### 27.6 Clean up (optional)

```bash
kubectl delete namespace svc-test
```

---

## 28. Removing kube-proxy

Once Cilium was healthy, we removed the kube-proxy DaemonSet:

```bash
kubectl -n kube-system delete ds kube-proxy
```

Then:

```bash
kubectl -n kube-system delete cm kube-proxy
```

The first command removes the running kube-proxy agents.

The second removes the configuration object.

---

## 29. Cleaning kube-proxy iptables Rules

Deleting the DaemonSet does not automatically mean every old kube-proxy iptables rule disappears.

We therefore cleaned the rules on every Kubernetes node.

### Cleanup script

```bash
# scripts/cleanup-kube-proxy.sh
#!/bin/bash
set -euo pipefail

echo "Cleaning kube-proxy iptables rules on $(hostname)..."

# Backup current rules
sudo iptables-save > /tmp/iptables-backup-$(date +%s).rules

# Remove KUBE-* rules
sudo iptables-save | grep -v KUBE | sudo iptables-restore

echo "Done. Backup saved to /tmp/iptables-backup-*.rules"
```

Run on each node:

```bash
# On cp01
ssh ubuntu@cp01.faekcorp.lab 'bash -s' < scripts/cleanup-kube-proxy.sh

# On cp02
ssh ubuntu@cp02.faekcorp.lab 'bash -s' < scripts/cleanup-kube-proxy.sh

# On cp03
ssh ubuntu@cp03.faekcorp.lab 'bash -s' < scripts/cleanup-kube-proxy.sh
```

Or manually on each node:

```bash
sudo iptables-save | grep -v KUBE | sudo iptables-restore
```

This removes the old `KUBE-*` iptables rules that kube-proxy created.

We did this on the Kubernetes nodes, not on the bastion.

---

## 30. Final kube-proxy Verification

From the bastion:

```bash
kubectl get ds -A
```

The expected final state is:

```
cilium        3   3   3
cilium-envoy  3   3   3
```

There should be no:

```
kube-proxy
```

We can also check:

```bash
kubectl -n kube-system get pods
```

There should be no:

```
kube-proxy-*
```

---

## 31. Final Architecture

The final dataplane is now:

```
                         Kubernetes API
                               |
                               |
                       haproxy.faekcorp.lab
                               |
                               v
                    ┌─────────────────────┐
                    │      HAProxy        │
                    │      :6443          │
                    └──────────┬──────────┘
                               |
                 ┌─────────────┼─────────────┐
                 |             |             |
                 v             v             v
               cp01          cp02          cp03
                 |             |             |
                 +-------------+-------------+
                               |
                          Cilium 1.20.1
                               |
                 ┌─────────────┴─────────────┐
                 |                           |
             eBPF datapath              Geneve tunnel
                 |                           |
                 v                           v
          Kubernetes Services          UDP 6081
                 |
                 v
             Pod network
             10.244.0.0/16
                 |
        ┌────────┴────────┐
        |                 |
     worker1           worker2
   10.70.61.85       10.70.11.67
```

There is now:

```
NO kube-proxy
NO kube-proxy iptables service programming
```

Cilium handles Kubernetes Service traffic using eBPF.

---

## 32. Joining Worker Nodes

### 32.1 Worker join architecture

Our architecture is:

```
                    Bastion
                 kubectl / helm
                       |
                       v
              HAProxy :6443
                       |
          ┌────────────┼────────────┐
          v            v            v
        cp01         cp02         cp03
          |            |            |
          └────────────┼────────────┘
                       |
                    Cilium
                       |
                 ┌─────┴─────┐
                 v           v
              worker1     worker2
```

The workers will **not** become control-plane nodes.

They will receive:

- kubelet
- kubeadm
- containerd
- Cilium
- Kubernetes node configuration

But they will **not** have:

- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager

And we intentionally do **not** install `kubectl` on the workers.

### 32.2 Generate a fresh worker join command

Instead of manually constructing the token, let kubeadm generate the complete command.

SSH to **cp01**:

```bash
ssh ubuntu@cp01.faekcorp.lab
```

Then run:

```bash
sudo kubeadm token create --print-join-command
```

You should get something similar to:

```text
kubeadm join haproxy.faekcorp.lab:6443 \
  --token xxxxxxxxxxxxxxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Why do this?

Because kubeadm will generate:

1. a fresh bootstrap token
2. the correct API endpoint
3. the correct CA discovery hash

So don't reuse an old token if it has expired.

**Important:** By default, kubeadm bootstrap tokens have a **24-hour TTL**. The CA hash does not expire in the same way; the token is the short-lived part.

### 32.3 Understand the join command

The generated command will look like:

```bash
kubeadm join haproxy.faekcorp.lab:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Each part has a purpose.

#### API endpoint

```
haproxy.faekcorp.lab:6443
```

The worker does **not** join directly to cp01.

It joins through our HA endpoint:

```
worker
   |
   v
haproxy.faekcorp.lab:6443
   |
   +---- cp01
   +---- cp02
   +---- cp03
```

That's important for HA.

#### Token

```
--token <TOKEN>
```

This is the temporary bootstrap credential that allows the new node to participate in the join process.

#### CA hash

```
--discovery-token-ca-cert-hash sha256:<HASH>
```

This protects against the worker being tricked into joining an API server controlled by an attacker.

Conceptually:

```
Worker
  |
  | "Is this really my Kubernetes cluster?"
  |
  v
API server certificate
  |
  v
CA public key/hash verification
  |
  v
Trusted Kubernetes cluster
```

### 32.4 Before joining worker1

Let's first make sure worker1 has the prerequisites we already prepared.

SSH into worker1:

```bash
ssh ubuntu@worker1
```

Then verify:

```bash
containerd --version
kubeadm version
kubelet --version
```

And:

```bash
sudo systemctl status containerd --no-pager
sudo systemctl status kubelet --no-pager
```

The kubelet may show `inactive` or waiting before the join. **That by itself is not a problem.**

The important prerequisite is that containerd is installed and running and the Kubernetes packages are installed.

### 32.5 Join worker1

Take the **fresh command generated on cp01**:

```bash
sudo kubeadm token create --print-join-command
```

Copy the entire output and run it on **worker1**.

It should look like:

```bash
sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <NEW_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

### ❌ Do not add:

```
--control-plane
```

This is a worker.

### 32.6 What happens during `kubeadm join`?

This is worth understanding.

The worker roughly goes through:

```
kubeadm join
      |
      v
Contact HAProxy
      |
      v
Reach Kubernetes API
      |
      v
Validate CA
      |
      v
Authenticate using bootstrap token
      |
      v
Download cluster information
      |
      v
Create kubelet configuration
      |
      v
Create kubelet certificates
      |
      v
Start kubelet
      |
      v
Node registers with API server
      |
      v
Cilium discovers the new node
```

At the end:

```
worker1
   |
   | kubelet
   |
   v
Kubernetes API
   |
   v
Node object created
```

### 32.7 Immediately check from the bastion

Once worker1 finishes joining, **do not install anything else yet**.

Go back to the bastion and run:

```bash
kubectl get nodes -o wide
```

We expect something similar to:

```
NAME                 STATUS   ROLES           INTERNAL-IP
cp01.faekcorp.lab    Ready    control-plane   10.70.21.6
cp02.faekcorp.lab    Ready    control-plane   10.70.31.209
cp03.faekcorp.lab    Ready    control-plane   10.70.41.250
worker1.faekcorp.lab Ready    <none>           10.70.61.85
```

**But don't worry if worker1 initially shows `NotReady`.**

That's actually an important Kubernetes concept.

The node can successfully register with the API before its networking is completely ready.

Since Cilium is our CNI, we need Cilium to recognize the new node.

### 32.8 Watch Cilium

From the bastion:

```bash
kubectl -n kube-system get pods -o wide -w
```

You should eventually see Cilium create a new agent on worker1:

```
cilium-xxxxx    1/1   Running   ...   worker1.faekcorp.lab
```

Why?

Cilium is a DaemonSet:

```
Cilium DaemonSet
       |
       +---- cp01
       +---- cp02
       +---- cp03
       +---- worker1
```

When a new Kubernetes node appears, the DaemonSet schedules a Cilium agent there.

### 32.9 Verify Cilium sees worker1

Run:

```bash
kubectl get ciliumnodes
```

We should eventually have:

```
cp01.faekcorp.lab
cp02.faekcorp.lab
cp03.faekcorp.lab
worker1.faekcorp.lab
```

This is an important milestone.

It means the worker isn't merely registered with Kubernetes — **Cilium has also registered the node into its networking model.**

### 32.10 Label the worker roles

By default, a worker joined with kubeadm doesn't get the nice role label:

```
node-role.kubernetes.io/worker
```

So let's add it ourselves.

From the bastion:

```bash
kubectl label node worker1.faekcorp.lab node-role.kubernetes.io/worker=worker
```

Then:

```bash
kubectl get nodes
```

Now you should see:

```
NAME                 STATUS   ROLES
cp01.faekcorp.lab    Ready    control-plane
cp02.faekcorp.lab    Ready    control-plane
cp03.faekcorp.lab    Ready    control-plane
worker1.faekcorp.lab Ready    worker
```

The label is:

```
node-role.kubernetes.io/worker=worker
```

The value itself isn't particularly important; the key is what Kubernetes tooling uses to display the role.

### 32.11 Then join worker2

Once worker1 is completely healthy, we'll do exactly the same thing for worker2.

Generate a fresh command again:

```bash
sudo kubeadm token create --print-join-command
```

You can actually reuse the same token if it hasn't expired, because the token can be used to join multiple nodes until its TTL expires.

But for learning purposes, generating a fresh command before worker2 is perfectly fine.

Then on worker2:

```bash
sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <NEW_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

Then from the bastion:

```bash
kubectl get nodes -o wide
kubectl get ciliumnodes
```

Then label it:

```bash
kubectl label node worker2.faekcorp.lab node-role.kubernetes.io/worker=worker
```

### 32.12 Final node layout

Our desired final cluster becomes:

```
                         BASTION
                    kubectl / helm
                           |
                           v
                  HAProxy + BIND9
                 10.70.11.192:6443
                           |
          ┌────────────────┼────────────────┐
          |                |                |
          v                v                v
        CP01             CP02             CP03
       10.70.21.6       10.70.31.209     10.70.41.250
          |                |                |
          └────────────────┼────────────────┘
                           |
                         Cilium
                           |
                ┌──────────┴──────────┐
                |                     |
                v                     v
             worker1               worker2
            10.70.61.85            10.70.11.67
```

And Kubernetes should report:

```
NAME                 STATUS   ROLES
cp01.faekcorp.lab    Ready    control-plane
cp02.faekcorp.lab    Ready    control-plane
cp03.faekcorp.lab    Ready    control-plane
worker1.faekcorp.lab Ready    worker
worker2.faekcorp.lab Ready    worker
```

### 32.13 Final verification

After both workers are joined:

```bash
kubectl get nodes -o wide
kubectl get ciliumnodes
kubectl get ds -A
kubectl -n kube-system get pods -o wide
```

Expected:

```
kube-system   cilium         5   5   5
kube-system   cilium-envoy   5   5   5
```

And 5 Cilium agents (cp01, cp02, cp03, worker1, worker2).

---

## 33. Worker Join Troubleshooting — The HAProxy Security Group Issue

### 33.1 The symptom

When we first attempted to join worker1, `kubeadm join` hung at:

```
[preflight] Running pre-flight checks
```

It did not proceed, and it did not error out — it simply stopped.

Meanwhile, on worker1:

```bash
sudo systemctl status kubelet.service
```

showed:

```
Active: activating (auto-restart) (Result: exit-code)
Main PID: 1905 (code=exited, status=1/FAILURE)
```

### 33.2 First diagnostic step — verify DNS

We checked:

```bash
sudo getent hosts haproxy.faekcorp.lab
```

Result:

```
10.70.11.192    haproxy.faekcorp.lab
```

✅ DNS was working.

### ❌ WRONG command

We initially ran:

```bash
sudo getent haproxy.faekcorp.lab
```

Result:

```
Unknown database: haproxy.faekcorp.lab
```

### ✅ CORRECT command

```bash
getent hosts haproxy.faekcorp.lab
```

The `hosts` argument is required. Without it, `getent` interprets the name as a database name.

### 33.3 Second diagnostic step — verify containerd

```bash
sudo systemctl status containerd.service -n 10 --no-pager
```

Result:

```
Active: active (running) since Sun 2026-10-04 16:31:51 UTC; 14min ago
```

✅ containerd was running.

### 33.4 Third diagnostic step — the kubelet status

```bash
sudo systemctl status kubelet.service -n 10 --no-pager
```

Result:

```
Active: activating (auto-restart) (Result: exit-code)
Main PID: 2007 (code=exited, status=1/FAILURE)
```

This looked alarming, but as we established:

> Before `kubeadm join`, kubelet commonly has nothing useful to run because kubeadm has not created its configuration yet. This is **not** necessarily the join problem.

### 33.5 The actual root cause — missing Security Group rule

The real issue was that the **HAProxy Security Group (`lb-dns-sg`) did not have an inbound rule allowing TCP 6443 from the worker Security Group (`worker-sg`)**.

Because of this:

```
worker1
   |
   | TCP 6443
   v
haproxy.faekcorp.lab (10.70.11.192)
   |
   ✗ BLOCKED — no inbound rule
```

`kubeadm join` could not complete its preflight checks because it could not reach the API server through HAProxy.

### 33.6 The fix

We added the missing inbound rule to the HAProxy Security Group:

| Source SG | Destination SG | Protocol | Port | Purpose |
|-----------|---------------|----------|------|---------|
| worker-sg | lb-dns-sg | TCP | 6443 | Worker → HAProxy API access |

After adding this rule, `kubeadm join` completed successfully.

### 33.7 Why this was not the same problem as Section 21

In Section 21, we discussed how the TLS error proved that the Security Group was **not** the problem — packets were reaching the API server.

But here in Section 33, the situation was different:

```
Section 21 (Cilium → API):
  Packets reached the API server.
  TLS rejected the certificate.
  → Security Group was fine.

Section 33 (worker → HAProxy):
  Packets never reached HAProxy.
  kubeadm hung at preflight.
  → Security Group was the problem.
```

The lesson is:

> The same architectural layer (network path to the API) can fail in different ways. The **error signature** tells you which layer to investigate.

| Symptom | Layer | Action |
|---------|-------|--------|
| `x509: certificate is valid for X, not Y` | TLS | Check certificate SANs |
| `i/o timeout` | Network/SG | Check Security Groups, routing |
| `kubeadm join` hangs at preflight | Network/SG | Check worker → HAProxy TCP 6443 |
| `connection refused` | Service | Check if API server is listening |

### 33.8 The successful worker join

Once the Security Group rule was added:

On **worker1**:

```bash
sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <NEW_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

Result:

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```

On **worker2**, the same procedure:

```bash
sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <NEW_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

Result:

```
This node has joined the cluster.
```

### 33.9 Verify the final node state

From the bastion:

```bash
kubectl get nodes
```

Result:

```
NAME                   STATUS   ROLES           AGE     VERSION
cp01.faekcorp.lab      Ready    control-plane   24h     v1.35.8
cp02.faekcorp.lab      Ready    control-plane   21h     v1.35.8
cp03.faekcorp.lab      Ready    control-plane   21h     v1.35.8
worker1.faekcorp.lab   Ready    worker          9m48s   v1.35.8
worker2.faekcorp.lab   Ready    worker          4m38s   v1.35.8
```

### 33.10 Verify Cilium on all 5 nodes

```bash
kubectl get ds -A
```

Result:

```
NAMESPACE     NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
kube-system   cilium         5         5         5       5            5
kube-system   cilium-envoy   5         5         5       5            5
```

```bash
kubectl get ciliumnodes
```

Result:

```
NAME                   CILIUMINTERNALIP   INTERNALIP
cp01.faekcorp.lab      ...
cp02.faekcorp.lab      ...
cp03.faekcorp.lab      ...
worker1.faekcorp.lab   ...
worker2.faekcorp.lab   ...
```

---

## 34. Mistakes Made During the Build

### 34.1 Mistake 1 — kube-proxy was still installed

**What happened:**

We enabled:

```
kubeProxyReplacement=true
```

but kubeadm had already installed:

```
kube-proxy
```

**Why?**

Because the cluster was not initialized with:

```bash
--skip-phases=addon/kube-proxy
```

**Lesson:**

For a brand-new kubeadm cluster intended to be kube-proxy-free:

```bash
kubeadm init \
  --skip-phases=addon/kube-proxy
```

For an existing cluster, perform a controlled migration instead.

---

### 34.2 Mistake 2 — Cilium Used the HAProxy IP for HTTPS

**What happened:**

We configured:

```
k8sServiceHost=10.70.11.192
```

Cilium connected to:

```
https://10.70.11.192:6443
```

but the API certificate did not contain:

```
IP Address:10.70.11.192
```

**Result:**

```
x509: certificate is valid for ...
not 10.70.11.192
```

**Fix:**

Use:

```
k8sServiceHost=haproxy.faekcorp.lab
```

because:

```
DNS:haproxy.faekcorp.lab
```

already existed in the certificate SAN.

---

### 34.3 Mistake 3 — Initially Interpreting the API Failure as a Networking Problem

The first symptom looked like:

```
dial tcp 10.96.0.1:443: i/o timeout
```

It would have been easy to immediately start modifying:

```
Security Groups
iptables
routes
Cilium tunnels
```

Instead we investigated layer by layer.

We checked:

```bash
kubectl get endpoints
```

Then:

```bash
kubectl describe svc kubernetes
```

Then Cilium logs.

Eventually the TLS error gave us the decisive evidence.

**Lesson:**

Do not troubleshoot Kubernetes networking by randomly changing Security Groups.

Identify the failing layer:

```
DNS
 ↓
Routing
 ↓
Security Group
 ↓
TCP
 ↓
TLS
 ↓
HTTP/API
 ↓
Kubernetes
```

Our failure had reached the TLS layer.

---

### 34.4 Mistake 4 — Missing Security Group Rule for Worker → HAProxy

**What happened:**

The HAProxy Security Group (`lb-dns-sg`) did not have an inbound rule allowing TCP 6443 from `worker-sg`.

`kubeadm join` hung at:

```
[preflight] Running pre-flight checks
```

**Why?**

Because the worker could not reach the Kubernetes API through HAProxy.

**Fix:**

Add the rule:

| Source SG | Destination SG | Protocol | Port |
|-----------|---------------|----------|------|
| worker-sg | lb-dns-sg | TCP | 6443 |

**Lesson:**

Before joining workers, verify that the worker Security Group can reach the HAProxy Security Group on TCP 6443.

---

## 35. Rollback Procedure

If the migration fails and Cilium cannot handle Services, kube-proxy can be restored.

### 35.1 Reinstall kube-proxy

**Option A — Via kubeadm phase:**

```bash
sudo kubeadm init phase addon kube-proxy \
  --config /etc/kubernetes/kubeadm-config.yaml
```

**Option B — Via kubectl apply:**

If you have a backup of the kube-proxy DaemonSet and ConfigMap:

```bash
kubectl apply -f backup/kube-proxy-ds.yaml
kubectl apply -f backup/kube-proxy-cm.yaml
```

### 35.2 Verify kube-proxy is running

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy
kubectl get ds -A | grep kube-proxy
```

### 35.3 Disable Cilium kube-proxy replacement

```bash
helm upgrade cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=false
```

### 35.4 Verify Services work again

```bash
kubectl -n svc-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://nginx.svc-test.svc.cluster.local
```

### 35.5 Backup before migration

Before removing kube-proxy, always back up:

```bash
mkdir -p backup
kubectl -n kube-system get ds kube-proxy -o yaml > backup/kube-proxy-ds.yaml
kubectl -n kube-system get cm kube-proxy -o yaml > backup/kube-proxy-cm.yaml
sudo iptables-save > backup/iptables-$(hostname)-$(date +%s).rules
```

### 35.6 Rollback a worker join

If a worker join fails and you want to reset the worker:

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /etc/kubernetes/
sudo iptables -F
sudo iptables -t nat -F
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Then re-run the join command.

---

## 36. Troubleshooting Commands Worth Keeping

### Check Cilium pods

```bash
kubectl -n kube-system get pods -o wide
```

### Check Cilium DaemonSet

```bash
kubectl -n kube-system get ds cilium
```

### Check Cilium status

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- \
  cilium-dbg status
```

### Check Cilium nodes

```bash
kubectl get ciliumnodes
```

### Check Cilium Helm configuration

```bash
helm get values cilium -n kube-system
```

### Check Kubernetes API Service

```bash
kubectl describe svc kubernetes
```

### Check API endpoints

```bash
kubectl get endpoints
```

For modern Kubernetes, EndpointSlice is preferred:

```bash
kubectl get endpointslices
```

### Check kube-proxy

```bash
kubectl get ds -A
kubectl -n kube-system get pods -l k8s-app=kube-proxy
```

### Inspect API certificate

```bash
sudo openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout \
  -subject \
  -issuer \
  -ext subjectAltName
```

### Inspect certificate through HAProxy

```bash
openssl s_client \
  -connect 10.70.11.192:6443 \
  -showcerts </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -ext subjectAltName
```

### Check worker → HAProxy reachability

```bash
nc -vz haproxy.faekcorp.lab 6443
curl -vk https://haproxy.faekcorp.lab:6443/version
```

### Check containerd CRI

```bash
sudo crictl info
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock info
sudo ctr plugins ls | grep -E 'io.containerd.grpc.v1.cri|cri'
```

### Check kubelet logs

```bash
sudo journalctl -u kubelet -n 50 --no-pager
```

### Check kubeadm join progress

```bash
sudo journalctl -u kubelet -f
```

### Check DNS resolution

```bash
getent hosts haproxy.faekcorp.lab
```

### Run Cilium connectivity test

```bash
cilium connectivity test
```

### Run Cilium status via CLI

```bash
cilium status --wait
```

### Observe Hubble flows

```bash
cilium hubble port-forward &
hubble observe --follow
```

---

## 37. Final Verification Checklist

Before declaring the migration successful:

### 37.1 All nodes Ready

```bash
kubectl get nodes
```

All nodes should be:

```
Ready
```

Expected:

```
NAME                   STATUS   ROLES
cp01.faekcorp.lab      Ready    control-plane
cp02.faekcorp.lab      Ready    control-plane
cp03.faekcorp.lab      Ready    control-plane
worker1.faekcorp.lab   Ready    worker
worker2.faekcorp.lab   Ready    worker
```

### 37.2 System pods healthy

```bash
kubectl get pods -A
```

### 37.3 Cilium healthy

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- \
  cilium-dbg status
```

Verify:

```
Kubernetes:              Ok
KubeProxyReplacement:    True
Cilium:                  Ok
Routing:                 Network: Tunnel [geneve]
Cluster health:          5/5 reachable
```

### 37.4 kube-proxy gone

```bash
kubectl get ds -A
```

No:

```
kube-proxy
```

### 37.5 Cilium nodes registered

```bash
kubectl get ciliumnodes
```

Should show 5 nodes.

### 37.6 API endpoints exist

```bash
kubectl get endpoints
```

### 37.7 ClusterIP Service works

```bash
kubectl -n svc-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://nginx.svc-test.svc.cluster.local
```

### 37.8 Cilium connectivity test

```bash
cilium connectivity test
```

---

## 38. Important Interview Concepts

### Why does Cilium need the Kubernetes API?

Cilium watches Kubernetes resources such as:

```
Pods
Services
Endpoints / EndpointSlices
NetworkPolicies
CiliumNetworkPolicies
Nodes
```

It uses this information to program its datapath.

---

### Why can we remove kube-proxy?

Normally kube-proxy implements Kubernetes Service networking using mechanisms such as:

```
iptables
IPVS
```

Cilium's kube-proxy replacement implements Service load-balancing using eBPF.

Therefore:

```
kube-proxy
```

is no longer required for Kubernetes Service handling.

---

### What does `KubeProxyReplacement: True` mean?

It means Cilium has enabled its replacement datapath for Kubernetes Services traditionally handled by kube-proxy.

It does not simply mean:

> Cilium is installed.

Cilium can be installed while kube-proxy still exists.

---

### Why do we need `k8sServiceHost`?

Cilium needs to know where the Kubernetes API server is reachable.

In our HA architecture, the correct endpoint is:

```
haproxy.faekcorp.lab:6443
```

rather than an individual control-plane IP.

That preserves the HA design:

```
Cilium
   |
   v
HAProxy
   |
   +---- cp01
   +---- cp02
   +---- cp03
```

If cp01 fails, the API endpoint does not need to change.

---

### What is the difference between Cilium and kube-proxy?

| Feature | kube-proxy | Cilium |
|---------|-----------|--------|
| Dataplane | iptables/IPVS | eBPF |
| Service handling | iptables rules | eBPF maps |
| Network policy | No | Yes |
| Observability | Limited | Hubble |
| Performance | Good | Better at scale |
| kube-proxy replacement | N/A | Yes |

---

### Why do workers use `haproxy.faekcorp.lab:6443` and not a specific control-plane IP?

Because the HAProxy endpoint provides high availability:

```
worker
   |
   v
haproxy.faekcorp.lab:6443
   |
   +---- cp01:6443
   +---- cp02:6443
   +---- cp03:6443
```

If any single control-plane node fails, the worker still has a working API endpoint.

If the worker joined directly to `cp01:6443` and cp01 failed, the worker would lose its API connection.

---

### Why does `kubeadm join` hang at preflight?

`kubeadm join` performs preflight checks including:

- Network connectivity to the API server
- DNS resolution
- containerd availability
- CRI responsiveness

If the worker cannot reach the API endpoint (for example, because of a Security Group block), `kubeadm join` can hang at preflight without producing an explicit error.

**Diagnostic step:** From the worker, run:

```bash
nc -vz haproxy.faekcorp.lab 6443
curl -vk https://haproxy.faekcorp.lab:6443/version
```

If these fail, the problem is at the network/Security Group layer, not at the Kubernetes layer.

---

## 39. Key Lessons From This Lab

### Lesson 1 — HA endpoint consistency matters

We configured:

```
controlPlaneEndpoint:
  haproxy.faekcorp.lab:6443
```

Therefore clients should preferably use:

```
haproxy.faekcorp.lab
```

rather than bypassing HAProxy with an individual node IP.

---

### Lesson 2 — TLS certificates are part of networking

Networking is not only:

```
IP
Route
Port
Security Group
```

For HTTPS connections, we also have:

```
Certificate
SAN
Hostname verification
Trust chain
```

A connection can successfully reach port 6443 and still fail because of TLS identity validation.

---

### Lesson 3 — Read the exact error

This:

```
i/o timeout
```

is different from:

```
connection refused
```

which is different from:

```
x509: certificate is valid for X, not Y
```

Each points to a different troubleshooting layer.

---

### Lesson 4 — Don't remove kube-proxy before Cilium is healthy

The correct sequence is:

```
Install Cilium
       ↓
Configure Cilium
       ↓
Fix API connectivity
       ↓
Fix TLS
       ↓
Cilium healthy
       ↓
KPR=True
       ↓
Cluster health 3/3
       ↓
Prove ClusterIP works
       ↓
Remove kube-proxy
       ↓
Clean old iptables rules
       ↓
Prove ClusterIP still works
```

---

### Lesson 5 — Existing clusters can migrate

We did not rebuild the cluster.

We migrated:

```
kube-proxy + Cilium
```

to:

```
Cilium only
```

by removing kube-proxy after validating that Cilium's replacement datapath was ready.

---

### Lesson 6 — Layer-based troubleshooting

Each layer has a distinct error signature. Identify the layer, and the fix becomes obvious.

```
DNS
 ↓
Routing
 ↓
Security Group
 ↓
TCP
 ↓
TLS
 ↓
HTTP/API
 ↓
Kubernetes
```

---

### Lesson 7 — Worker join requires correct Security Group rules

Joining a worker requires:

```
worker
   |
   | TCP 6443
   v
haproxy.faekcorp.lab
```

If the HAProxy Security Group does not allow inbound TCP 6443 from the worker Security Group, `kubeadm join` hangs at preflight.

**Always verify:**

```bash
nc -vz haproxy.faekcorp.lab 6443
curl -vk https://haproxy.faekcorp.lab:6443/version
```

before troubleshooting deeper.

---

### Lesson 8 — DaemonSets automatically deploy to new nodes

Cilium is a DaemonSet. When a new worker node joins:

```
New node appears
       |
       v
DaemonSet controller notices
       |
       v
Schedules Cilium agent on the new node
       |
       v
Cilium registers the node in its datapath
```

We do not manually install Cilium on workers.

---

## 40. Final State

The final Kubernetes networking architecture is:

```
                    Kubernetes HA API
                           |
                           v
                haproxy.faekcorp.lab:6443
                           |
          ┌────────────────┼────────────────┐
          |                |                |
         cp01             cp02             cp03
          |                |                |
          └────────────────┼────────────────┘
                           |
                       Cilium 1.20.1
                           |
              ┌────────────┴────────────┐
              |                         |
           eBPF                      Geneve
       Service LB                  UDP 6081
              |                         |
              └────────────┬────────────┘
                           |
                    Pod network
                   10.244.0.0/16
                           |
                  ┌────────┴────────┐
                  |                 |
               worker1           worker2
             10.70.61.85       10.70.11.67
```

Final Cilium configuration:

```
Cilium version:             1.20.1
Kubernetes:                 1.35.8

IPAM:
  mode:                     cluster-pool
  pool:                     10.244.0.0/16
  per-node allocation:      /24

Routing:
  mode:                     tunnel
  tunnel protocol:          Geneve
  tunnel port:              UDP 6081

Kubernetes API:
  host:                     haproxy.faekcorp.lab
  port:                     6443

Kube-proxy replacement:
  enabled:                  true

kube-proxy:
  removed

Nodes:
  control-plane:            cp01, cp02, cp03
  workers:                  worker1, worker2
  cilium agents:            5 (one per node)
```

The key end result is:

```
                     BEFORE

                Kubernetes Service
                       |
                 kube-proxy
                       |
                    iptables
                       |
                    Pod


                     AFTER

                Kubernetes Service
                       |
                 Cilium eBPF
                       |
                    Pod
```

**Cilium is now the CNI and the Kubernetes Service dataplane, while kube-proxy has been removed. Five nodes are healthy: three control-plane nodes and two workers.**

---

## Appendix A — Hubble Observability

Cilium was installed with Hubble enabled. Hubble provides flow-level observability.

### Enable Hubble CLI port-forward

```bash
cilium hubble port-forward &
```

### Observe flows

```bash
hubble observe --follow
```

### Observe flows for a specific namespace

```bash
hubble observe --namespace svc-test --follow
```

### Observe dropped packets

```bash
hubble observe --verdict DROPPED --follow
```

### Check Hubble status

```bash
cilium status | grep Hubble
```

Expected:

```
Hubble:   Ok   Current/Max Flows: 4095/4095 (100.00%), Flows/s: 12.34
```

---

---

## Appendix B — References

- [Cilium Documentation](https://docs.cilium.io/)
- [Cilium Kubernetes Compatibility Matrix](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/)
- [Cilium kube-proxy Replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Cilium Helm Chart](https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium)
- [Kubernetes kubeadm Documentation](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Kubernetes TLS Bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
- [kubeadm Token Documentation](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)

---
