# Kubernetes HA — Cilium eBPF / kube-proxy Replacement

**Complete — Installation, Troubleshooting, and Migration**

---

## ⚠️ Reading This Document

This document intentionally shows:

- ✅ **CORRECT** commands
- ❌ **WRONG** commands we ran
- 🔧 **FIXES** we applied
- ⚠️ **INCOMPLETE** places where we stopped and had to investigate further

When something is marked ❌ **WRONG** or ⚠️ **INCOMPLETE**, the correct version follows immediately after.

This mirrors the actual troubleshooting journey, which is the most valuable part for future projects and interviews.

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
worker1.faekcorp.lab (joined later)
worker2.faekcorp.lab (joined later)
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
```

The Kubernetes API endpoint is:

```text
haproxy.faekcorp.lab:6443
```

HAProxy distributes API traffic to:

```text
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

```text
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

```text
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

```text
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

```text
cluster-pool
```

with:

```text
10.244.0.0/16
```

Cilium divides this pool into smaller per-node blocks.

For example, our nodes received:

```text
cp01 → 10.244.0.0/24
cp02 → 10.244.1.0/24
cp03 → 10.244.2.0/24
```

We verified this with:

```bash
kubectl get ciliumnodes
```

Result:

```text
NAME                CILIUMINTERNALIP   INTERNALIP
cp01.faekcorp.lab   10.244.0.140       10.70.21.6
cp02.faekcorp.lab   10.244.1.187       10.70.31.209
cp03.faekcorp.lab   10.244.2.193       10.70.41.250
```

The important point is that the `CILIUMINTERNALIP` is from the Kubernetes pod network while `INTERNALIP` is the EC2/node network.

---

## 7. Geneve Networking

We selected:

```text
routingMode=tunnel
tunnelProtocol=geneve
```

Geneve encapsulates pod traffic between nodes.

Conceptually:

```text
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

```text
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

**Troubleshooting lesson:** Do not assume a Security Group is the problem simply because a pod is not healthy. Use the actual error to determine the failing layer.

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

```text
dial tcp 10.96.0.1:443: i/o timeout
```

The address:

```text
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

```text
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

```text
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

```text
cilium        3   3   3
cilium-envoy  3   3   3
kube-proxy    3   3   3
```

And:

```bash
kubectl -n kube-system get pods
```

showed:

```text
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

Because this cluster was already running, we chose to perform a controlled migration instead of rebuilding the cluster. This is documented in Sections 23–26.

---

## 13. Cilium API Endpoint Configuration

For kube-proxy replacement, Cilium needs a reliable Kubernetes API endpoint.

We configured:

```text
k8sServiceHost
k8sServicePort
```

### ❌ WRONG initial value

We initially used the HAProxy IP:

```text
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

```text
Establishing connection to apiserver
ipAddr=https://10.70.11.192:6443
```

followed by:

```text
tls: failed to verify certificate:
x509: certificate is valid for 10.96.0.1, 10.70.31.209,
not 10.70.11.192
```

This error was extremely valuable.

It told us:

```text
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

```text
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

```text
DNS:haproxy.faekcorp.lab
```

was present.

But:

```text
IP Address:10.70.11.192
```

was NOT present.

Therefore:

```text
https://10.70.11.192:6443
```

could not pass certificate hostname/IP verification.

But:

```text
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

```text
haproxy.faekcorp.lab
```

The mistake was not in the kubeadm configuration.

The mistake was that we later configured Cilium to access the API using:

```text
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

```text
DNS:haproxy.faekcorp.lab
```

but no:

```text
IP Address:10.70.11.192
```

This confirmed that HAProxy was not modifying the certificate.

HAProxy was simply forwarding the Kubernetes API TLS connection.

---

## 18. Root Cause

The complete chain was:

```text
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

```text
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

```text
apiserver.crt
```

because the certificate already contained:

```text
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

```text
Certificate:
DNS:haproxy.faekcorp.lab
```

Valid:

```text
https://haproxy.faekcorp.lab:6443
```

Not valid:

```text
https://10.70.11.192:6443
```

unless the certificate also contains:

```text
IP Address:10.70.11.192
```

### General troubleshooting rule

When you see:

```text
x509:
certificate is valid for X,
not Y
```

think:

```text
TLS identity mismatch
```

not:

```text
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

---

## 21. Security Group Troubleshooting

The cluster also required Geneve networking.

Cilium Geneve uses:

```text
UDP 6081
```

The node Security Groups therefore allowed UDP 6081 between the control-plane and worker node groups.

Relevant Cilium health traffic also included:

```text
TCP 4240
```

and Kubernetes control-plane traffic included:

```text
TCP 6443
```

The important troubleshooting lesson was:

> Do not assume a Security Group is the problem simply because a pod is not healthy.

We used the actual error to determine the layer that was failing.

The TLS error:

```text
x509: certificate is valid for ...
not ...
```

proved that packets were reaching the API server far enough to perform TLS negotiation.

Therefore we investigated certificates instead of blindly changing Security Groups.

---

## 22. Verifying the Cilium Configuration

We checked:

```bash
kubectl get cm cilium-config -n kube-system -o yaml
```

Important values included:

```text
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

```text
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

```text
Kubernetes:              Ok
```

This means Cilium can communicate with the Kubernetes API.

```text
KubeProxyReplacement:    True
```

This means Cilium's kube-proxy replacement is enabled.

```text
Routing:                 Network: Tunnel [geneve]
```

This confirms Geneve tunneling.

```text
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

```text
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

```text
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

```text
kube-system   cilium-operator   1/1
kube-system   coredns            2/2
```

---

## 26. Migration Strategy

We deliberately did not immediately delete kube-proxy.

The migration sequence was:

```text
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
14. Final verification
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

```text
cilium        3   3   3
cilium-envoy  3   3   3
```

There should be no:

```text
kube-proxy
```

We can also check:

```bash
kubectl -n kube-system get pods
```

There should be no:

```text
kube-proxy-*
```

---

## 31. Final Architecture

The final dataplane is now:

```text
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
```

There is now:

```text
NO kube-proxy
NO kube-proxy iptables service programming
```

Cilium handles Kubernetes Service traffic using eBPF.

---

## 32. Mistakes Made During the Build

### 32.1 Mistake 1 — kube-proxy was still installed

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

### 32.2 Mistake 2 — Cilium Used the HAProxy IP for HTTPS

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

### 32.3 Mistake 3 — Initially Interpreting the API Failure as a Networking Problem

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

## 33. Rollback Procedure

If the migration fails and Cilium cannot handle Services, kube-proxy can be restored.

### 33.1 Reinstall kube-proxy

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

### 33.2 Verify kube-proxy is running

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy
kubectl get ds -A | grep kube-proxy
```

### 33.3 Disable Cilium kube-proxy replacement

```bash
helm upgrade cilium cilium/cilium \
  --version 1.20.1 \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=false
```

### 33.4 Verify Services work again

```bash
kubectl -n svc-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://nginx.svc-test.svc.cluster.local
```

### 33.5 Backup before migration

Before removing kube-proxy, always back up:

```bash
mkdir -p backup
kubectl -n kube-system get ds kube-proxy -o yaml > backup/kube-proxy-ds.yaml
kubectl -n kube-system get cm kube-proxy -o yaml > backup/kube-proxy-cm.yaml
sudo iptables-save > backup/iptables-$(hostname)-$(date +%s).rules
```

---

## 34. Troubleshooting Commands Worth Keeping

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

## 35. Final Verification Checklist

Before declaring the migration successful:

### 35.1 All nodes Ready

```bash
kubectl get nodes
```

All nodes should be:

```
Ready
```

### 35.2 System pods healthy

```bash
kubectl get pods -A
```

### 35.3 Cilium healthy

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
Cluster health:          3/3 reachable
```

### 35.4 kube-proxy gone

```bash
kubectl get ds -A
```

No:

```
kube-proxy
```

### 35.5 Cilium nodes registered

```bash
kubectl get ciliumnodes
```

### 35.6 API endpoints exist

```bash
kubectl get endpoints
```

### 35.7 ClusterIP Service works

```bash
kubectl -n svc-test run client --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://nginx.svc-test.svc.cluster.local
```

### 35.8 Cilium connectivity test

```bash
cilium connectivity test
```

---

## 36. Important Interview Concepts

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

## 37. Key Lessons From This Lab

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

## 38. Final State

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

**Cilium is now the CNI and the Kubernetes Service dataplane, while kube-proxy has been removed.**

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

## Appendix B — Repository Structure

```
cilium-kube-proxy-replacement/
├── README.md                          # This document
├── values/
│   └── cilium-values.yaml             # Reproducible install
├── manifests/
│   └── svc-test.yaml                  # ClusterIP test workload
├── scripts/
│   ├── cleanup-kube-proxy.sh          # iptables cleanup
│   ├── verify-cilium.sh               # Health checks
│   └── rollback-kube-proxy.sh         # Rollback procedure
└── docs/
    ├── troubleshooting.md             # TLS/SAN deep dive
    └── architecture.md                # Diagrams
```

---

## Appendix C — References

- [Cilium Documentation](https://docs.cilium.io/)
- [Cilium Kubernetes Compatibility Matrix](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/)
- [Cilium kube-proxy Replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Cilium Helm Chart](https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium)
- [Kubernetes kubeadm Documentation](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Kubernetes TLS Bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)

---
