# Production-Ready Kubernetes Cluster with kubeadm: Complete Documentation

## Multi-AZ, Private Nodes, Bastion Host, Calico CNI, VXLAN

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Infrastructure Components](#infrastructure-components)
3. [Network Design](#network-design)
4. [Security Group Design](#security-group-design)
5. [Kubernetes Components](#kubernetes-components)
6. [Calico CNI Architecture](#calico-cni-architecture)
7. [Port Reference Guide](#port-reference-guide)
8. [Deployment Steps](#deployment-steps)
9. [Troubleshooting Guide](#troubleshooting-guide)
10. [HA Evolution Path](#ha-evolution-path)
11. [Security Best Practices](#security-best-practices)

---

## 1. Architecture Overview

### 1.1 High-Level Architecture

```
                         INTERNET
                            │
                     Internet Gateway
                            │
        ┌───────────────────┴────────────────────┐
        │                                        │
        ▼                                        ▼
   PUBLIC SUBNETS                          NAT GATEWAY
        │                                        │
        │                                        │
   ┌────▼─────┐                                  │
   │ BASTION  │                                  │
   └────┬─────┘                                  │
        │ SSH + kubectl                          │
        │                                        │
════════╪════════════════════════════════════════╪════════
        │                                        │
        │          PRIVATE KUBERNETES NETWORK    │
        │                                        │
        ▼                                        ▼
   ┌───────────┐                          Outbound Internet
   │ Control   │                                 │
   │ Plane     │                                 │
   │ AZ-1      │                                 │
   └───────────┘                                 │
                                                 │
              ┌──────────────────────────────────┘
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
  Worker 1         Worker 2              Worker 3
    AZ-1             AZ-2                  AZ-3
```

### 1.2 Key Characteristics

| Feature | Implementation |
|---------|---------------|
| **High Availability** | Nodes distributed across 3 Availability Zones |
| **Security** | Private nodes, bastion host as single entry point |
| **Cost Optimization** | Single NAT Gateway for outbound internet |
| **Production-Oriented** | Production-style VPC design with private/public subnets |
| **CNI** | Calico with VXLAN overlay (BGP disabled) |
| **Container Runtime** | containerd with systemd cgroup driver |
| **Kubernetes Version** | v1.32+ |

---

## 2. Infrastructure Components

### 2.1 AWS Infrastructure Components

| Component | Purpose | Location |
|-----------|---------|----------|
| **VPC** | Network isolation | 10.50.0.0/16 |
| **Internet Gateway** | VPC ↔ Internet connectivity | Attached to VPC |
| **NAT Gateway** | Private subnets → Internet (outbound) | Public subnet AZ-1 |
| **Bastion Host** | Administrative jump box, kubectl | Public subnet AZ-1 |
| **Control Plane** | Kubernetes control plane (kubeadm) | Private subnet AZ-1 |
| **Worker 1** | Kubernetes worker node | Private subnet AZ-1 |
| **Worker 2** | Kubernetes worker node | Private subnet AZ-2 |
| **Worker 3** | Kubernetes worker node | Private subnet AZ-3 |

### 2.2 EC2 Instance Specifications

| Node Type | vCPU | RAM | Disk | OS |
|-----------|------|-----|------|-----|
| **Bastion** | 2 | 4 GB | 20 GB | Ubuntu 22.04/24.04 LTS |
| **Control Plane** | 2+ | 4-8 GB | 20 GB | Ubuntu 22.04/24.04 LTS |
| **Worker** | 1-2 | 2-4 GB | 10-20 GB | Ubuntu 22.04/24.04 LTS |

> **Important**: kubeadm requires minimum 2 vCPU for control plane. Use `--ignore-preflight-errors=NumCPU` only for demos.

---

## 3. Network Design

### 3.1 VPC CIDR

```
VPC: 10.50.0.0/16
Total IPs: 65,536
```

### 3.2 Subnet Design

#### Public Subnets (For Bastion + NAT)

| AZ | Subnet | Purpose |
|----|--------|---------|
| AZ-1 | 10.50.1.0/24 | Bastion Host + NAT Gateway |
| AZ-2 | 10.50.2.0/24 | Future public resources |
| AZ-3 | 10.50.3.0/24 | Future public resources |

#### Private Subnets (For Kubernetes Nodes)

| AZ | Subnet | Nodes |
|----|--------|-------|
| AZ-1 | 10.50.11.0/24 | Control Plane + Worker 1 |
| AZ-2 | 10.50.12.0/24 | Worker 2 |
| AZ-3 | 10.50.13.0/24 | Worker 3 |

### 3.3 Route Tables

#### Public Route Table

| Destination | Target |
|-------------|--------|
| 10.50.0.0/16 | local |
| 0.0.0.0/0 | Internet Gateway |

**Associated Subnets**: All public subnets (10.50.1.0/24, 10.50.2.0/24, 10.50.3.0/24)

#### Private Route Table

| Destination | Target |
|-------------|--------|
| 10.50.0.0/16 | local |
| 0.0.0.0/0 | NAT Gateway (in AZ-1) |

**Associated Subnets**: All private subnets (10.50.11.0/24, 10.50.12.0/24, 10.50.13.0/24)

### 3.4 Network Topology Diagram

```
┌───────────────────────────────────────────────────────────────┐
│                         AWS VPC                               │
│                       10.50.0.0/16                            │
│                                                               │
│   ┌──────────────────────────────────────────────────────┐    │
│   │                  INTERNET GATEWAY                    │    │
│   └───────────────────────┬──────────────────────────────┘    │
│                           │                                   │
│   ┌───────────────────────┴───────────────────────────────┐   │
│   │                    PUBLIC SUBNETS                      │   │
│   │                                                        │   │
│   │ AZ-1: 10.50.1.0/24                                    │   │
│   │                                                        │   │
│   │    ┌──────────────┐       ┌──────────────┐             │   │
│   │    │   BASTION    │       │ NAT GATEWAY  │             │   │
│   │    └──────┬───────┘       └──────┬───────┘             │   │
│   │           │                      │                     │   │
│   └───────────┼──────────────────────┼─────────────────────┘   │
│               │                      │                         │
│═══════════════╪══════════════════════╪═════════════════════════│
│               │                      │                         │
│                 PRIVATE SUBNETS                                │
│                                                               │
│   AZ-1                  AZ-2                   AZ-3           │
│                                                               │
│ 10.50.11.0/24        10.50.12.0/24        10.50.13.0/24       │
│                                                               │
│ ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│ │ Control Plane│      │   Worker 2   │      │   Worker 3   │  │
│ │              │      │              │      │              │  │
│ │ Worker 1     │      │              │      │              │  │
│ └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 4. Security Group Design

### 4.1 Security Group Overview

We use three Security Groups:

```text
1. bastion-sg      → Administration entry point
2. control-plane-sg → Kubernetes control plane protection
3. worker-sg       → Worker node protection
```

### 4.2 `bastion-sg`

**Purpose**: Administrative entry point - very restricted

#### Inbound Rules

| Priority | Source | Protocol | Port | Purpose |
|----------|--------|----------|------|---------|
| 1 | YOUR_PUBLIC_IP/32 | TCP | 22 | SSH access from your IP only |

#### Outbound Rules

| Priority | Destination | Protocol | Port | Purpose |
|----------|-------------|----------|------|---------|
| 1 | 0.0.0.0/0 | ALL | ALL | Internet access for updates, private node access |

> **Security Note**: Replace `YOUR_PUBLIC_IP/32` with your actual IP. Never use `0.0.0.0/0` for SSH.

### 4.3 `control-plane-sg`

**Purpose**: Protect Kubernetes control plane components

#### Inbound Rules

| Priority | Source | Protocol | Port | Component | Purpose |
|----------|--------|----------|------|-----------|---------|
| 1 | `bastion-sg` | TCP | 22 | SSH | Administrative SSH |
| 2 | `bastion-sg` | TCP | 6443 | kube-apiserver | kubectl access |
| 3 | `worker-sg` | TCP | 6443 | kube-apiserver | Worker registration |
| 4 | `worker-sg` | UDP | 4789 | VXLAN | Calico overlay traffic |
| 5 | `worker-sg` | TCP | 5473 | Typha | Felix → Typha (if Typha on CP) |
| 6 | `control-plane-sg` | UDP | 4789 | VXLAN | Self VXLAN (if CP runs calico-node) |
| 7 | `control-plane-sg` | TCP | 5473 | Typha | Self Typha (if Typha on CP) |

#### Outbound Rules

| Priority | Destination | Protocol | Port | Purpose |
|----------|-------------|----------|------|---------|
| 1 | 0.0.0.0/0 | ALL | ALL | Internet access (via NAT) |

### 4.4 `worker-sg`

**Purpose**: Protect worker nodes

#### Inbound Rules

| Priority | Source | Protocol | Port | Component | Purpose |
|----------|--------|----------|------|-----------|---------|
| 1 | `bastion-sg` | TCP | 22 | SSH | Administrative SSH |
| 2 | `control-plane-sg` | TCP | 10250 | kubelet | Control plane → kubelet |
| 3 | `control-plane-sg` | UDP | 4789 | VXLAN | Calico overlay from CP |
| 4 | `worker-sg` | UDP | 4789 | VXLAN | Calico overlay between workers |
| 5 | `control-plane-sg` | TCP | 5473 | Typha | Felix → Typha (if Typha on workers) |
| 6 | `worker-sg` | TCP | 5473 | Typha | Felix → Typha (if Typha on workers) |

#### Outbound Rules

| Priority | Destination | Protocol | Port | Purpose |
|----------|-------------|----------|------|---------|
| 1 | 0.0.0.0/0 | ALL | ALL | Internet access (via NAT) |

### 4.5 Security Group Rule Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SECURITY GROUP COMMUNICATION MATRIX                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SOURCE ───────────────────────────────────────────► DESTINATION            │
│                                                                             │
│  YOUR_IP/32  ── TCP 22 ──► bastion-sg                                      │
│                                                                             │
│  bastion-sg  ── TCP 22 ──► control-plane-sg                                │
│  bastion-sg  ── TCP 6443 ─► control-plane-sg                               │
│  bastion-sg  ── TCP 22 ──► worker-sg                                       │
│                                                                             │
│  worker-sg   ── TCP 6443 ─► control-plane-sg                               │
│  worker-sg   ── UDP 4789 ─► control-plane-sg                               │
│  worker-sg   ── TCP 5473 ─► control-plane-sg (if Typha on CP)              │
│                                                                             │
│  control-plane-sg ─ TCP 10250 ─► worker-sg                                 │
│  control-plane-sg ─ UDP 4789 ─► worker-sg                                  │
│  control-plane-sg ─ TCP 5473 ─► worker-sg (if Typha on workers)            │
│                                                                             │
│  worker-sg   ── UDP 4789 ─► worker-sg                                      │
│  worker-sg   ── TCP 5473 ─► worker-sg (if Typha on workers)                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Kubernetes Components

### 5.1 Component Overview

| Component | Purpose | Runs On | Port |
|-----------|---------|---------|------|
| **kube-apiserver** | Kubernetes API front door | Control Plane | TCP 6443 |
| **etcd** | Distributed key-value store (cluster state) | Control Plane | TCP 2379 (client), TCP 2380 (peer) |
| **kube-scheduler** | Pod placement decisions | Control Plane | Internal |
| **kube-controller-manager** | Reconciliation loops | Control Plane | Internal |
| **kubelet** | Node agent, container management | All Nodes | TCP 10250 |
| **kube-proxy** | Services networking | All Nodes | Internal |
| **containerd** | Container runtime | All Nodes | Unix socket |
| **CoreDNS** | Cluster DNS | System Pods | UDP/TCP 53 (internal) |

### 5.2 Component Communication Diagram

```
                    ┌─────────────────────────────────────┐
                    │         YOUR LAPTOP                 │
                    └────────────────┬────────────────────┘
                                     │
                                     │ SSH (22)
                                     ▼
                    ┌─────────────────────────────────────┐
                    │          BASTION HOST               │
                    │         (kubectl installed)         │
                    └────────────────┬────────────────────┘
                                     │
                      ┌──────────────┼──────────────┐
                      │              │              │
                SSH (22)       kubectl (6443)      │
                      │              │              │
                      ▼              ▼              ▼
                    ┌─────────────────────────────────────┐
                    │         CONTROL PLANE               │
                    │                                     │
                    │  ┌─────────────────────────────┐    │
                    │  │     kube-apiserver :6443     │    │
                    │  ├─────────────────────────────┤    │
                    │  │           etcd :2379         │    │
                    │  ├─────────────────────────────┤    │
                    │  │      kube-scheduler          │    │
                    │  ├─────────────────────────────┤    │
                    │  │   kube-controller-manager    │    │
                    │  └─────────────────────────────┘    │
                    └────────────────┬────────────────────┘
                                     │
                              10250 (kubelet)
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │    WORKER 1     │  │    WORKER 2     │  │    WORKER 3     │
    │                 │  │                 │  │                 │
    │   kubelet:10250 │  │   kubelet:10250 │  │   kubelet:10250 │
    │   containerd    │  │   containerd    │  │   containerd    │
    └─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## 6. Calico CNI Architecture

### 6.1 Calico Components

| Component | Purpose | Runs As | Port |
|-----------|---------|---------|------|
| **Felix** | Node-level networking/policy agent | `calico-node` DaemonSet | N/A (programs kernel) |
| **Typha** | Scaling/distribution layer for Felix | Deployment | TCP 5473 |
| **calico-kube-controllers** | Syncs Kubernetes ↔ Calico state | Deployment | N/A |
| **Tigera Operator** | Manages Calico installation | Deployment | N/A |
| **CNI Plugin** | Configures Pod network interfaces | Binary | N/A |

### 6.2 Calico Architecture Diagram

```
                       KUBERNETES API
                              │
                              │
                  ┌───────────┴───────────┐
                  │                       │
                  ▼                       ▼
          Tigera Operator        calico-kube-controllers
                  │                       │
                  │                       │
                  │                 reconciles state
                  │                       │
                  └───────────┬───────────┘
                              │
                              ▼
                            Typha
                       TCP 5473
                              │
                ┌─────────────┼─────────────┐
                │             │             │
                ▼             ▼             ▼
             Felix          Felix         Felix
             Node 1         Node 2        Node 3
                │             │             │
                │             │             │
                ▼             ▼             ▼
             Linux          Linux         Linux
            networking     networking    networking
                │             │             │
                └──────┬──────┴──────┬──────┘
                       │               │
                       │ VXLAN         │
                       │ UDP 4789      │
                       ▼               ▼
                    Pod network across nodes
```

### 6.3 Calico Configuration

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled                    # We use VXLAN, not BGP
    ipPools:
    - cidr: 192.168.0.0/16           # Pod CIDR
      natOutgoing: Enabled           # SNAT for pod→external traffic
      blockSize: 26                  # 64 IPs per node
      encapsulation: VXLANCrossSubnet # VXLAN overlay
      nodeSelector: all()
```

### 6.4 Why VXLAN vs BGP?

| Aspect | VXLAN | BGP |
|--------|-------|-----|
| **Encapsulation** | Yes (UDP 4789) | No (native routing) |
| **Port Required** | UDP 4789 | TCP 179 |
| **AWS Compatibility** | Excellent | Requires route propagation |
| **Complexity** | Lower | Higher |
| **Performance** | Slight overhead | Native performance |
| **Use Case** | Our choice for lab | Large-scale on-prem |

---

## 7. Port Reference Guide

### 7.1 Ports We Open (Required)

| Port | Protocol | Component | Why | Direction |
|------|----------|-----------|-----|-----------|
| **22** | TCP | SSH | Administration access | Inbound to all nodes |
| **6443** | TCP | kube-apiserver | Kubernetes API | Inbound to CP |
| **10250** | TCP | kubelet | Node management | Inbound to workers |
| **4789** | UDP | VXLAN | Pod networking overlay | Inbound to all nodes |
| **5473** | TCP | Typha | Calico config distribution | Inbound to Typha destination |

### 7.2 Ports We Don't Open (Not Required)

| Port | Protocol | Component | Why Not |
|------|----------|-----------|---------|
| **179** | TCP | BGP | We use VXLAN, BGP disabled |
| **2379** | TCP | etcd client | Single CP lab (add for HA) |
| **2380** | TCP | etcd peer | Single CP lab (add for HA) |
| **10257** | TCP | controller-manager | Not a node-to-node traffic path |
| **10259** | TCP | scheduler | Not a node-to-node traffic path |
| **53** | TCP/UDP | CoreDNS | Internal Kubernetes service |
| **30000-32767** | TCP/UDP | NodePort | Not exposing publicly |
| **containerd** | Unix socket | containerd | Local communication only |

### 7.3 Decision Tree for Opening Ports

```
Component has a listening port?
    │
    YES
    │
    ▼
Does traffic from another machine need to reach this port?
    │
    NO → ❌ Don't open in SG
    │       (Local Unix socket, internal controller, etc.)
    │
    YES
    │
    ▼
Which machine(s) initiate the connection?
    │
    ├── Bastion → Component → Add INBOUND rule from bastion-sg
    │
    ├── Workers → Component → Add INBOUND rule from worker-sg
    │
    ├── Control Plane → Component → Add INBOUND rule from control-plane-sg
    │
    └── Component ↔ Component → Add INBOUND rule to both SGs
```

---

## 8. Deployment Steps

### 8.1 Pre-requisites Checklist

- [ ] AWS Account with VPC permissions
- [ ] SSH Key Pair created in AWS
- [ ] Security Groups created
- [ ] 5 EC2 instances (1 Bastion, 1 CP, 3 Workers)
- [ ] Ubuntu 22.04/24.04 LTS AMI
- [ ] All instances in same VPC
- [ ] Private subnets have route to NAT Gateway

### 8.2 Network Setup Steps

1. **Create VPC**
   ```
   VPC CIDR: 10.50.0.0/16
   ```

2. **Create Subnets**
   ```
   Public Subnet AZ-1:  10.50.1.0/24
   Public Subnet AZ-2:  10.50.2.0/24
   Public Subnet AZ-3:  10.50.3.0/24
   Private Subnet AZ-1: 10.50.11.0/24
   Private Subnet AZ-2: 10.50.12.0/24
   Private Subnet AZ-3: 10.50.13.0/24
   ```

3. **Create Internet Gateway**
   - Attach to VPC

4. **Create NAT Gateway**
   - Deploy in Public Subnet AZ-1
   - Assign Elastic IP

5. **Create Route Tables**
   - **Public RT**: 0.0.0.0/0 → Internet Gateway
   - **Private RT**: 0.0.0.0/0 → NAT Gateway

6. **Create Security Groups**
   - `bastion-sg`
   - `control-plane-sg`
   - `worker-sg`

### 8.3 Instance Launch Steps

| Instance | Subnet | Security Group | Private IP (example) |
|----------|--------|---------------|---------------------|
| Bastion | Public AZ-1 | bastion-sg | 10.50.1.10 |
| Control Plane | Private AZ-1 | control-plane-sg | 10.50.11.10 |
| Worker 1 | Private AZ-1 | worker-sg | 10.50.11.20 |
| Worker 2 | Private AZ-2 | worker-sg | 10.50.12.10 |
| Worker 3 | Private AZ-3 | worker-sg | 10.50.13.10 |

### 8.4 Kubernetes Installation Steps

#### Step 1: Disable Swap & Set Kernel Networking [ALL NODES]

```bash
# Disable swap (required by kubelet)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules & sysctls for container networking
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

#### Step 2: Install containerd [ALL NODES]

```bash
# Install containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Align containerd with kubelet expectations
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

#### Step 3: Install kubeadm, kubelet, kubectl [ALL NODES]

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
 | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

#### Step 4: Initialize Control Plane [CONTROL PLANE ONLY]

```bash
# Replace with your control plane's private IP
sudo kubeadm init \
  --control-plane-endpoint=10.50.11.10:6443 \
  --apiserver-advertise-address=10.50.11.10 \
  --pod-network-cidr=192.168.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Step 5: Install Calico CNI [CONTROL PLANE ONLY]

```bash
# Install the operator and CRDs
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

# Apply Installation CR with VXLAN and BGP disabled
cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - cidr: 192.168.0.0/16
      natOutgoing: Enabled
      blockSize: 26
      encapsulation: VXLANCrossSubnet
      nodeSelector: all()
EOF
```

#### Step 6: Join Workers [WORKER NODES]

```bash
# On Control Plane - get join command
kubeadm token create --print-join-command

# On each worker - run the printed command
sudo kubeadm join 10.50.11.10:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

#### Step 7: Configure kubectl on Bastion

```bash
# On Bastion
sudo apt-get update && sudo apt-get install -y kubectl

# Copy kubeconfig from Control Plane
scp ubuntu@10.50.11.10:/etc/kubernetes/admin.conf ~/.kube/config

# Verify cluster access
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 9. Troubleshooting Guide

### 9.1 Common Issues and Solutions

#### Issue: `calico-node` Pods Show `Running (0/1)`

**Symptom**: `calico-node` pods running but not ready

**Root Cause**: Typha connection timeout or BGP issues

**Solution**:

```bash
# Check BGP setting
kubectl get installation.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'

# Disable BGP if using VXLAN
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'

# Restart calico-node
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node
```

#### Issue: Workers Can't Join Cluster

**Symptom**: `kubeadm join` fails with connection refused

**Check**:
1. Verify Security Group rules allow TCP 6443 from worker-sg to control-plane-sg
2. Verify control plane is running: `kubectl get nodes`
3. Check API server is listening: `netstat -tlnp | grep 6443`

#### Issue: Pods Can't Reach Other Pods

**Symptom**: Cross-node pod communication failing

**Check**:
1. Verify Security Group allows UDP 4789 between worker-sg and control-plane-sg
2. Check Calico pods are running: `kubectl -n calico-system get pods`
3. Verify VXLAN interface exists: `ip link show vxlan.calico`

#### Issue: NodePort Services Not Accessible

**Symptom**: Can't reach NodePort service from bastion

**Note**: We intentionally didn't open NodePort ports (30000-32767) in SG. If you need NodePort access:
1. Temporarily allow port range
2. Or use LoadBalancer/Ingress instead

### 9.2 Health Check Commands

```bash
# Check nodes
kubectl get nodes -o wide

# Check all system pods
kubectl get pods -A

# Check Calico status
kubectl -n calico-system get pods -o wide
kubectl -n calico-system get deployment,svc,endpoints

# Check control plane components
kubectl get componentstatuses

# Check API server endpoint
kubectl cluster-info

# Check DNS
kubectl run test-pod --rm -it --image=busybox -- nslookup kubernetes.default.svc.cluster.local
```

### 9.3 Reset Commands

```bash
# On Worker Nodes
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd

# On Control Plane
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd /etc/cni/net.d /var/lib/cni /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd
```

---

## 10. HA Evolution Path

### 10.1 Current vs HA Architecture

| Aspect | Current Lab | HA Production |
|--------|-------------|---------------|
| **Control Planes** | 1 | 3 |
| **etcd** | Single | 3-node cluster |
| **Availability** | CP single point of failure | HA across AZs |
| **API Server** | 1 endpoint | Load balanced |
| **Cost** | Lower | Higher |

### 10.2 HA Architecture Diagram

```
                        LOAD BALANCER
                              │
                    ┌─────────┼─────────┐
                    │         │         │
                    ▼         ▼         ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │   CP1   │ │   CP2   │ │   CP3   │
              │  AZ-1   │ │  AZ-2   │ │  AZ-3   │
              └────┬────┘ └────┬────┘ └────┬────┘
                   │           │           │
                   └─────etcd──┼───────────┘
                        2379/2380
                   │           │           │
                   ▼           ▼           ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │ Worker1 │ │ Worker2 │ │ Worker3 │
              │  AZ-1   │ │  AZ-2   │ │  AZ-3   │
              └─────────┘ └─────────┘ └─────────┘
```

### 10.3 Additional SG Rules for HA

When upgrading to HA, add these rules to `control-plane-sg`:

| Priority | Source | Protocol | Port | Purpose |
|----------|--------|----------|------|---------|
| 1 | `control-plane-sg` | TCP | 2379 | etcd client communication |
| 2 | `control-plane-sg` | TCP | 2380 | etcd peer communication |

### 10.4 HA Deployment Steps Summary

1. **Add etcd SG rules** (2379/2380 between CPs)
2. **Create Load Balancer** for API server (TCP 6443)
3. **Initialize first CP** with `--control-plane-endpoint=<LB_IP>`
4. **Join additional CPs** with `--control-plane`
5. **Configure etcd** as cluster (not standalone)
6. **Update bastion kubectl** to use load balancer endpoint

---

## 11. Security Best Practices

### 11.1 Security Group Best Practices

| Practice | Implementation |
|----------|---------------|
| **Principle of Least Privilege** | Only open ports that are absolutely needed |
| **Restricted SSH** | Only bastion has public SSH (from your IP) |
| **No Direct Internet Access** | Private nodes use NAT for outbound only |
| **SG as Source** | Use SGs as sources, not IP ranges |
| **Stateful Awareness** | Leverage SG statefulness for return traffic |

### 11.2 Networking Best Practices

| Practice | Implementation |
|----------|---------------|
| **Private Subnets** | All Kubernetes nodes in private subnets |
| **NAT Gateway** | Single NAT for outbound internet (cost-optimized) |
| **VPC Design** | Clean separation: 10.50.1-3.x (public), 10.50.11-13.x (private) |
| **Route Tables** | Public → IGW, Private → NAT |

### 11.3 Kubernetes Security Best Practices

| Practice | Implementation |
|----------|---------------|
| **RBAC** | Implement least-privilege RBAC policies |
| **Network Policies** | Use Calico NetworkPolicy for pod-level security |
| **Pod Security Standards** | Apply Pod Security Standards |
| **Secrets Management** | Use external secrets management (e.g., HashiCorp Vault) |
| **Regular Updates** | Keep Kubernetes and OS patched |

### 11.4 Calico Security Best Practices

| Practice | Implementation |
|----------|---------------|
| **Network Policies** | Default deny all ingress/egress, allow only required traffic |
| **Micro-segmentation** | Namespace-level network policies |
| **BGP Disabled** | For VXLAN-only environments, disable BGP |
| **Typha** | Enable Typha for production scaling |

### 11.5 Cost Optimization

| Practice | Implementation |
|----------|---------------|
| **Single NAT** | One NAT Gateway for all private subnets |
| **Reserved Instances** | Use reserved instances for production |
| **EBS Optimization** | Use gp3 volumes, not gp2 |
| **Delete on Termination** | Set DeleteOnTermination for root volumes |

---

## 12. Quick Reference Cards

### 12.1 Security Groups Quick Reference

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SECURITY GROUPS QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  bastion-sg:                                                               │
│    Inbound:  TCP 22 from YOUR_IP/32                                        │
│    Outbound: ALL to 0.0.0.0/0                                              │
│                                                                             │
│  control-plane-sg:                                                         │
│    Inbound:  TCP 22 from bastion-sg                                        │
│    Inbound:  TCP 6443 from bastion-sg, worker-sg                          │
│    Inbound:  UDP 4789 from worker-sg, control-plane-sg                    │
│    Inbound:  TCP 5473 from worker-sg, control-plane-sg (if Typha)         │
│    Outbound: ALL to 0.0.0.0/0                                              │
│                                                                             │
│  worker-sg:                                                                │
│    Inbound:  TCP 22 from bastion-sg                                        │
│    Inbound:  TCP 10250 from control-plane-sg                               │
│    Inbound:  UDP 4789 from worker-sg, control-plane-sg                    │
│    Inbound:  TCP 5473 from worker-sg, control-plane-sg (if Typha)         │
│    Outbound: ALL to 0.0.0.0/0                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Ports Quick Reference

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PORTS QUICK REFERENCE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PORT OPENED (5 total):                                                    │
│  ─────────────────────                                                      │
│  TCP 22      → SSH administration                                          │
│  TCP 6443    → Kubernetes API                                              │
│  TCP 10250   → Kubelet                                                     │
│  UDP 4789    → VXLAN (Calico overlay)                                      │
│  TCP 5473    → Typha (Calico config distribution)                          │
│                                                                             │
│  PORT NOT OPENED:                                                          │
│  ─────────────────                                                         │
│  TCP 179     → BGP (disabled - using VXLAN)                                │
│  TCP 2379    → etcd client (single CP lab)                                 │
│  TCP 2380    → etcd peer (single CP lab)                                   │
│  TCP 10257   → controller-manager                                          │
│  TCP 10259   → scheduler                                                   │
│  TCP/UDP 53  → CoreDNS (internal only)                                     │
│  TCP 30000+  → NodePort (not exposing)                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 12.3 Communication Flow Quick Reference

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       COMMUNICATION FLOW QUICK REFERENCE                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Your Laptop → Bastion:            TCP 22                                   │
│  Bastion → Control Plane:          TCP 22 (SSH), TCP 6443 (kubectl)        │
│  Bastion → Workers:               TCP 22 (SSH)                              │
│  Workers → Control Plane:         TCP 6443 (API registration)              │
│  Control Plane → Workers:         TCP 10250 (kubelet)                      │
│  All Nodes → All Nodes:           UDP 4789 (VXLAN overlay)                 │
│  Felix → Typha:                   TCP 5473 (config distribution)           │
│  Outbound Internet (Private):     Via NAT Gateway                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. References

### 13.1 Official Documentation

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubeadm Installation Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)
- [AWS Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)

### 13.2 Port References

- [Kubernetes Ports and Protocols](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Calico System Requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)
- [Calico Typha Configuration](https://docs.tigera.io/calico/latest/reference/typha/configuration)

### 13.3 Additional Resources

- [Calico VXLAN Configuration](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/config-options#use-vxlan)
- [kubeadm HA Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)

---

## 14. Conclusion

### 14.1 Design Principles Learned

1. **Ports have reasons** - Every port we open serves a specific purpose
2. **Security Groups protect destinations** - The SG of the receiver gets the inbound rule
3. **Security Groups are stateful** - Return traffic is automatically allowed
4. **Less is more** - Only open what's absolutely required
5. **Architecture first** - Understand the components before creating SGs

### 14.2 Key Takeaways

```text
✅ Private nodes with NAT for outbound internet
✅ Bastion host as single administration entry point
✅ Multi-AZ distribution for resilience
✅ Calico with VXLAN (BGP disabled) for Pod networking
✅ Only 5 ports open across all SGs
✅ Production-oriented design with cost optimization
✅ Clear upgrade path to HA
```

### 14.3 Future Enhancements

- [ ] Add 3 control planes for HA
- [ ] Configure etcd cluster (2379/2380)
- [ ] Add load balancer for API server
- [ ] Implement Calico NetworkPolicy for micro-segmentation
- [ ] Add monitoring with Prometheus/Grafana
- [ ] Implement logging with EFK/ELK stack
- [ ] Add Ingress Controller (NGINX/HAProxy)
- [ ] Configure SSL/TLS for API server
- [ ] Implement backup for etcd
- [ ] Set up CI/CD pipeline

---

## Appendix A: Terraform Example

```hcl
# Main VPC
resource "aws_vpc" "main" {
  cidr_block = "10.50.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "k8s-prod-vpc" }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "k8s-prod-igw" }
}

# Public Subnets
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.50.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet-${count.index + 1}" }
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.50.${count.index + 11}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = false
  tags = { Name = "private-subnet-${count.index + 1}" }
}

# NAT Gateway
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags          = { Name = "k8s-prod-nat" }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = { Name = "public-rt" }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  tags = { Name = "private-rt" }
}

# Security Groups
resource "aws_security_group" "bastion" {
  name        = "bastion-sg"
  description = "Bastion host security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP/32"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "control_plane" {
  name        = "control-plane-sg"
  description = "Control plane security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }
  
  ingress {
    from_port       = 6443
    to_port         = 6443
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id, aws_security_group.worker.id]
  }
  
  ingress {
    from_port       = 4789
    to_port         = 4789
    protocol        = "udp"
    security_groups = [aws_security_group.worker.id, aws_security_group.control_plane.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "worker" {
  name        = "worker-sg"
  description = "Worker node security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }
  
  ingress {
    from_port       = 10250
    to_port         = 10250
    protocol        = "tcp"
    security_groups = [aws_security_group.control_plane.id]
  }
  
  ingress {
    from_port       = 4789
    to_port         = 4789
    protocol        = "udp"
    security_groups = [aws_security_group.worker.id, aws_security_group.control_plane.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Appendix B: Compliance Checklist

### Security Compliance

- [ ] All nodes in private subnets
- [ ] Bastion host as single SSH entry point
- [ ] SG rules use SGs as sources (not IP ranges)
- [ ] SSH restricted to specific IP
- [ ] No public NodePort exposure
- [ ] VXLAN encryption (Calico with VXLAN)
- [ ] NAT Gateway for outbound internet only

### Operational Compliance

- [ ] All nodes have correct hostnames
- [ ] Swap disabled on all nodes
- [ ] containerd with systemd cgroup driver
- [ ] Kubernetes version pinned (no auto-updates)
- [ ] kubelet configured correctly
- [ ] Calico with VXLAN and BGP disabled
- [ ] Typha enabled for production scaling

### Monitoring Compliance

- [ ] Node monitoring enabled
- [ ] Control plane component monitoring
- [ ] Calico pod monitoring
- [ ] etcd monitoring
- [ ] Custom metrics for application pods

### Backup Compliance

- [ ] etcd backup configured
- [ ] Critical manifests backed up
- [ ] Persistent volume backups (if used)
- [ ] Disaster recovery plan documented

---

## Appendix C: Troubleshooting Commands Cheatsheet

```bash
# Node Status
kubectl get nodes -o wide
kubectl describe node <node-name>

# Pod Status
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>

# Calico Status
kubectl -n calico-system get pods -o wide
kubectl -n calico-system get deployment,svc,endpoints
kubectl -n calico-system logs ds/calico-node -c calico-node | grep -i typha
kubectl get installation.operator.tigera.io default -o yaml

# Network
ip route show
ip link show vxlan.calico
netstat -tlnp | grep -E "6443|10250|5473"

# System
sudo systemctl status containerd kubelet
journalctl -u kubelet -f
sudo kubeadm token list
sudo kubeadm certs renew all
```

---

## Document Metadata

| Field | Value |
|-------|-------|
| **Document Version** | 1.0 |
| **Kubernetes Version** | v1.32+ |
| **CNI** | Calico v3.30.2 |
| **Container Runtime** | containerd |
| **OS** | Ubuntu 22.04/24.04 LTS |
| **Cloud Provider** | AWS |
| **Last Updated** | 2026-09-09 |

---
