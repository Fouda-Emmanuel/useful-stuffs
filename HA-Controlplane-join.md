# Kubernetes HA Cluster — Control Plane Join Setup Documentation 

**Environment:** AWS EC2 · Ubuntu · Kubernetes v1.35.8 · containerd 2.2.6
**Cluster domain:** `faekcorp.lab`
**Control plane endpoint:** `haproxy.faekcorp.lab:6443`
**Status:** Control plane (cp01, cp02, cp03) initialized and joined. Cilium installation is documented separately.

---

## 1. Architecture Overview

### 1.1 Node Inventory

| Node | Role | IP | Components |
|---|---|---|---|
| `haproxy` | API LB + DNS | 10.70.11.192 | HAProxy, BIND9 |
| `bastion` | Admin workstation | — | kubectl 1.35.8 |
| `cp01` | Control plane | 10.70.21.6 | containerd, kubelet, kubeadm |
| `cp02` | Control plane | 10.70.31.209 | containerd, kubelet, kubeadm |
| `cp03` | Control plane | 10.70.41.250 | containerd, kubelet, kubeadm |
| `worker1` | Worker | 10.70.61.85 | containerd, kubelet, kubeadm |
| `worker2` | Worker | 10.70.11.67 | containerd, kubelet, kubeadm |

### 1.2 Component Versions

| Component | Version |
|---|---|
| Kubernetes (kubeadm, kubelet, kubectl) | v1.35.8 |
| containerd | 2.2.6 |
| crictl | 1.35.0 |
| kubeadm config API | `kubeadm.k8s.io/v1beta4` |
| kubelet config API | `kubelet.config.k8s.io/v1beta1` |

### 1.3 Network Layout

| Network | CIDR | Purpose |
|---|---|---|
| AWS VPC | 10.70.0.0/16 | Node IPs |
| Pod network | 10.244.0.0/16 | Pod IPs (CNI-managed) |
| Service network | 10.96.0.0/12 | Kubernetes Service VIPs |

### 1.4 AWS Security Groups

| SG | Purpose |
|---|---|
| `controlplane-sg` | cp01, cp02, cp03 |
| `worker-sg` | worker1, worker2 |
| `bastion-sg` | bastion |
| `lb-dns-sg` | HAProxy + BIND9 |

### 1.5 High-Level Diagram

```
                         ┌──────────────────┐
                         │     Bastion      │
                         │     kubectl      │
                         └────────┬─────────┘
                                  │
                                  │ TCP 6443
                                  ▼
                         ┌──────────────────┐
                         │     HAProxy      │
                         │    + BIND9       │
                         │  10.70.11.192    │
                         │     :6443        │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             ▼             ▼
                 cp01          cp02          cp03
              10.70.21.6   10.70.31.209   10.70.41.250
                 :6443         :6443          :6443
                    │             │             │
                    └─────────────┼─────────────┘
                                  │
                            etcd cluster
                            3 members
                            quorum = 2
```

---

## 2. DNS Configuration (BIND9 on HAProxy host)

### 2.1 Zone file

**File:** `/etc/bind/db.faekcorp.lab`

```bind
$TTL 86400
@ IN SOA haproxy.faekcorp.lab. admin.faekcorp.lab. (
    2025121701 ; Serial
    3600       ; Refresh
    1800       ; Retry
    604800     ; Expire
    86400      ; Negative Cache TTL
)

; NS record
@ IN NS haproxy.faekcorp.lab.

; A records
haproxy IN A 10.70.11.192
cp01    IN A 10.70.21.6
cp02    IN A 10.70.31.209
cp03    IN A 10.70.41.250
worker1 IN A 10.70.61.85
worker2 IN A 10.70.11.67
```

### 2.2 Verification

From any node:

```bash
getent hosts haproxy.faekcorp.lab
getent hosts cp01.faekcorp.lab
getent hosts cp02.faekcorp.lab
getent hosts cp03.faekcorp.lab
getent hosts worker1.faekcorp.lab
getent hosts worker2.faekcorp.lab
```

Expected:

```
10.70.11.192   haproxy.faekcorp.lab
10.70.21.6     cp01.faekcorp.lab
10.70.31.209   cp02.faekcorp.lab
10.70.41.250   cp03.faekcorp.lab
10.70.61.85    worker1.faekcorp.lab
10.70.11.67    worker2.faekcorp.lab
```

---

## 3. Pre-flight Verification on cp01

```bash
# Verify versions
kubeadm version
kubelet --version
sudo containerd --version
sudo crictl --version
```

Expected:

```
kubeadm version: &version.Info{... GitVersion:"v1.35.8" ...}
Kubernetes v1.35.8
containerd github.com/containerd/containerd v2.2.6
crictl version 1.35.0
```

```bash
# Verify CRI and services
sudo crictl info
sudo systemctl status containerd --no-pager
sudo systemctl status kubelet --no-pager
```

> kubelet may be inactive or restarting before `kubeadm init` — this is expected.

---

## 4. kubeadm Configuration File

**File:** `/root/kubeadm-ha.yaml` (created on cp01)

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

kubernetesVersion: v1.35.8

controlPlaneEndpoint: "haproxy.faekcorp.lab:6443"

networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

cgroupDriver: systemd
```

### 4.1 Field reference

| Field | Value | Meaning |
|---|---|---|
| `apiVersion` | `kubeadm.k8s.io/v1beta4` | kubeadm config API version (not Kubernetes version) |
| `kind` | `ClusterConfiguration` | Describes the cluster |
| `kubernetesVersion` | `v1.35.8` | Must match installed kubeadm/kubelet |
| `controlPlaneEndpoint` | `haproxy.faekcorp.lab:6443` | Stable HA API endpoint |
| `podSubnet` | `10.244.0.0/16` | Cluster Pod CIDR (CNI-managed) |
| `serviceSubnet` | `10.96.0.0/12` | Service VIP range |
| `cgroupDriver` | `systemd` | Matches containerd cgroup setup |

### 4.2 Create the file

```bash
sudo tee /root/kubeadm-ha.yaml >/dev/null <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

kubernetesVersion: v1.35.8

controlPlaneEndpoint: "haproxy.faekcorp.lab:6443"

networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

cgroupDriver: systemd
EOF

sudo cat /root/kubeadm-ha.yaml
```

---

## 5. HAProxy Configuration

**File:** `/etc/haproxy/haproxy.cfg` (on HAProxy host)

Relevant frontend/backend for Kubernetes API:

```haproxy
frontend k8s_api_frontend
    bind 10.70.11.192:6443
    mode tcp
    option tcplog
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    option tcp-check
    balance roundrobin
    server cp01 10.70.21.6:6443 check
    server cp02 10.70.31.209:6443 check
    server cp03 10.70.41.250:6443 check
```

Reload HAProxy after changes:

```bash
sudo systemctl reload haproxy
sudo systemctl status haproxy --no-pager
sudo ss -lntp | grep 6443
```

Expected:

```
LISTEN 0 4096 10.70.11.192:6443 ...
```

---

## 6. Initialize cp01

### 6.1 Optional dry run

```bash
sudo kubeadm init \
  --config=/root/kubeadm-ha.yaml \
  --upload-certs \
  --dry-run
```

### 6.2 Actual initialization

```bash
sudo kubeadm init \
  --config=/root/kubeadm-ha.yaml \
  --upload-certs \
  | sudo tee /root/kubeadm-init.out
```

Expected output (end):

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join haproxy.faekcorp.lab:6443 --token <TOKEN> \
        --discovery-token-ca-cert-hash sha256:<HASH>

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

kubeadm join haproxy.faekcorp.lab:6443 --token <TOKEN> \
        --discovery-token-ca-cert-hash sha256:<HASH> \
        --control-plane --certificate-key <CERT_KEY>
```

**Save the two join commands** (worker + control-plane). They are also in `/root/kubeadm-init.out`.

```bash
grep -A2 "kubeadm join" /root/kubeadm-init.out
```

### 6.3 Verify what kubeadm created

```bash
sudo ls -l /etc/kubernetes/
sudo ls -l /etc/kubernetes/pki/
sudo ls -l /etc/kubernetes/manifests/
```

Expected manifests:

```
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

### 6.4 Confirm the API server is up

```bash
sudo ss -lntp | grep 6443
curl -k https://127.0.0.1:6443/livez
```

Expected:

```
*:6443  ...  kube-apiserver
ok
```

---

## 7. Point cp01 kubelet at the local API server

This avoids the `cp01 → HAProxy → cp01` circular dependency.

```bash
# On cp01
sudo sed -i "s|https://haproxy.faekcorp.lab:6443|https://10.70.21.6:6443|" /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

Verify:

```bash
grep server /etc/kubernetes/kubelet.conf
# → server: https://10.70.21.6:6443
```

---

## 8. Configure kubectl on the bastion

The bastion is the only node that runs `kubectl`.

### 8.1 Copy admin.conf from cp01

```bash
# From cp01
scp /etc/kubernetes/admin.conf ubuntu@bastion:/tmp/admin.conf
```

### 8.2 Install kubeconfig on bastion

```bash
# On bastion
mkdir -p ~/.kube
sudo mv /tmp/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
```

### 8.3 Point bastion kubeconfig at HAProxy

```bash
# On bastion
kubectl config set-cluster kubernetes \
  --server=https://haproxy.faekcorp.lab:6443 \
  --kubeconfig=$HOME/.kube/config
```

### 8.4 Verify

```bash
# On bastion
kubectl get nodes
```

Expected:

```
NAME   STATUS     ROLES           AGE   VERSION
cp01   NotReady   control-plane   ...   v1.35.8
```

> `NotReady` is expected until CNI is installed.

---

## 9. AWS Security Groups — Required Rules

Both directions are required because HAProxy is a **TCP proxy** in the middle.

### 9.1 Rule: control planes → HAProxy

**On `lb-dns-sg`, inbound:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| Custom TCP | TCP | 6443 | `controlplane-sg` (sg-0174b378a646c08e4) |

### 9.2 Rule: HAProxy → control planes

**On `controlplane-sg`, inbound:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| Custom TCP | TCP | 6443 | `lb-dns-sg` (sg-018eb9861deb1b3b5) |

### 9.3 Verification from cp02

```bash
# On cp02
nc -vz -w 5 10.70.11.192 6443
curl -k --connect-timeout 5 https://haproxy.faekcorp.lab:6443/livez
```

Expected:

```
Connection to 10.70.11.192 6443 port [tcp/*] succeeded!
ok
```

---

## 10. Join cp02 as a control plane

### 10.1 Regenerate certificate key (if needed)

The `kubeadm-certs` Secret is temporary (~2h TTL). If the original key from `kubeadm init` expired:

```bash
# On cp01
sudo kubeadm init phase upload-certs --upload-certs
```

This prints a fresh `--certificate-key`.

### 10.2 Verify token is still valid

```bash
# On cp01
kubeadm token list
```

If expired or missing:

```bash
# On cp01
kubeadm token create --print-join-command
```

### 10.3 Run the join on cp02

```bash
# On cp02
sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY>
```

Expected milestones:

```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the "kubeadm-config" ConfigMap
[download-certs] Downloading the certificates in Secret "kubeadm-certs"
[etcd] Announced new etcd member joining to the existing etcd cluster
[mark-control-plane] Marking the node cp02 as control-plane
[kubelet-start] Starting the kubelet

This node has joined the cluster and a new control plane instance was created
```

### 10.4 Repoint cp02 kubelet at the local API server

```bash
# On cp02
sudo sed -i "s|https://haproxy.faekcorp.lab:6443|https://10.70.31.209:6443|" /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

Verify:

```bash
grep server /etc/kubernetes/kubelet.conf
# → server: https://10.70.31.209:6443
```

---

## 11. Join cp03 as a control plane

Same process as cp02, with cp03's values.

```bash
# On cp03 — verify DNS first
getent hosts haproxy.faekcorp.lab
```

Then join:

```bash
# On cp03
sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY>
```

Repoint kubelet:

```bash
# On cp03
sudo sed -i "s|https://haproxy.faekcorp.lab:6443|https://10.70.41.250:6443|" /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

Verify:

```bash
grep server /etc/kubernetes/kubelet.conf
# → server: https://10.70.41.250:6443
```

---

## 12. Verify the 3-node control plane

From the **bastion**:

```bash
kubectl get nodes
```

Expected:

```
NAME   STATUS     ROLES           AGE   VERSION
cp01   NotReady   control-plane   ...   v1.35.8
cp02   NotReady   control-plane   ...   v1.35.8
cp03   NotReady   control-plane   ...   v1.35.8
```

```bash
kubectl get pods -n kube-system -o wide | grep -E 'NAME|cp0'
```

Expected (all `Running`):

```
etcd-cp01
etcd-cp02
etcd-cp03
kube-apiserver-cp01
kube-apiserver-cp02
kube-apiserver-cp03
kube-controller-manager-cp01
kube-controller-manager-cp02
kube-controller-manager-cp03
kube-scheduler-cp01
kube-scheduler-cp02
kube-scheduler-cp03
```

### 12.1 Verify etcd quorum

```bash
kubectl -n kube-system exec etcd-cp01 -- \
  etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

Expected: 3 members, all `started`.

```bash
kubectl -n kube-system exec etcd-cp01 -- \
  etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster
```

Expected: all three endpoints `healthy: true`.

---

## 13. Join the workers

### 13.1 worker1

```bash
# On worker1
getent hosts haproxy.faekcorp.lab

sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### 13.2 worker2

```bash
# On worker2
getent hosts haproxy.faekcorp.lab

sudo kubeadm join haproxy.faekcorp.lab:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### 13.3 Verify

```bash
# On bastion
kubectl get nodes
```

Expected (all `NotReady` until CNI is installed):

```
NAME      STATUS     ROLES           AGE   VERSION
cp01      NotReady   control-plane   ...   v1.35.8
cp02      NotReady   control-plane   ...   v1.35.8
cp03      NotReady   control-plane   ...   v1.35.8
worker1   NotReady   <none>          ...   v1.35.8
worker2   NotReady   <none>          ...   v1.35.8
```

---

## 14. Troubleshooting Log (Real Incidents)

### Incident 1 — cp02 could not reach HAProxy on TCP 6443

**Symptom:** `kubeadm join` hung at `[preflight]`.

**Diagnostics:**

```bash
# DNS — worked
getent hosts haproxy.faekcorp.lab
# → 10.70.11.192 haproxy.faekcorp.lab

# TCP — timed out
nc -vz -w 5 10.70.11.192 6443
# → nc: connect to 10.70.11.192 port 6443 timed out

# HAProxy listener — up
sudo ss -lntp | grep 6443
# → LISTEN 10.70.11.192:6443

# HAProxy → cp01 — worked
nc -vz 10.70.21.6 6443
# → succeeded

# cp01 API — healthy
curl -k https://127.0.0.1:6443/livez
# → ok
```

**Root cause:** `lb-dns-sg` was missing an inbound rule allowing `controlplane-sg` to reach TCP 6443.

**Why cp01 worked but cp02 didn't:**

- `cp01` ran `kubeadm init` — it created its own API server **locally** and did not need to reach HAProxy.
- `cp02` ran `kubeadm join` — it had to reach the existing cluster **through HAProxy**, which required the missing SG rule.

**Fix:** Added inbound rule to `lb-dns-sg`:

| Type | Protocol | Port | Source |
|---|---|---|---|
| Custom TCP | TCP | 6443 | `controlplane-sg` (sg-0174b378a646c08e4) |

**Verification after fix:**

```bash
# From cp02
nc -vz -w 5 10.70.11.192 6443
# → succeeded

curl -k --connect-timeout 5 https://haproxy.faekcorp.lab:6443/livez
# → ok
```

---

### Incident 2 — kubeadm-certs Secret not found

**Symptom:** After fixing networking, join progressed to `[download-certs]` and failed:

```
error downloading certs:
Secret "kubeadm-certs" was not found
```

kubeadm also advised:

```
Please run
kubeadm init phase upload-certs --upload-certs
on a control plane
to generate a new one
```

**Root cause:** The `kubeadm-certs` Secret is temporary (default TTL ~2 hours). The original certificate key from `kubeadm init` had expired.

**Fix (on cp01):**

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

This generated a fresh certificate key. The join command was rerun with the new key.

> We did **not** manually create the Secret, scp certificates, or edit certificate files. kubeadm manages certificate distribution.

---

### Incident 3 — Token expired (if encountered)

**Fix (on cp01):**

```bash
kubeadm token create --print-join-command
```

---

## 15. Credentials and Secrets — Quick Reference

### Join command components

| Flag | Purpose |
|---|---|
| `--token` | Bootstrap authentication (valid 24h) |
| `--discovery-token-ca-cert-hash sha256:...` | Verify cluster CA identity |
| `--control-plane` | Join as control plane (not worker) |
| `--certificate-key` | Retrieve uploaded control-plane certificates |

### kubeadm certificate flow

```
             cp01
              │
              │ kubeadm init --upload-certs
              ▼
       kubeadm-certs Secret
       (namespace: kube-system)
              │
              │ --certificate-key
              ▼
       cp02 / cp03
       (download control-plane certs)
```

### Certificate upload regeneration

```bash
# On an existing control plane
sudo kubeadm init phase upload-certs --upload-certs
```

### Token regeneration

```bash
# On an existing control plane
kubeadm token create --print-join-command
```

---

## 16. Verification Checklist

### 16.1 DNS

```bash
getent hosts haproxy.faekcorp.lab
```

### 16.2 TCP reachability

```bash
nc -vz -w 5 haproxy.faekcorp.lab 6443
```

### 16.3 HAProxy listener

```bash
# On HAProxy
sudo ss -lntp | grep 6443
```

### 16.4 HAProxy → backend

```bash
# On HAProxy
nc -vz 10.70.21.6 6443
nc -vz 10.70.31.209 6443
nc -vz 10.70.41.250 6443
```

### 16.5 API health via LB

```bash
curl -k --connect-timeout 5 https://haproxy.faekcorp.lab:6443/livez
```

### 16.6 Control-plane pods

```bash
kubectl get pods -n kube-system -o wide | grep -E 'NAME|cp0'
```

### 16.7 etcd members

```bash
kubectl -n kube-system exec etcd-cp01 -- \
  etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### 16.8 Nodes

```bash
kubectl get nodes
```

---

## 17. Troubleshooting Checklist for Future Control-Plane Joins

Check in this order:

1. **DNS**
   ```bash
   getent hosts haproxy.faekcorp.lab
   ```

2. **TCP reachability**
   ```bash
   nc -vz -w 5 haproxy.faekcorp.lab 6443
   ```

3. **HAProxy listener**
   ```bash
   sudo ss -lntp | grep 6443
   ```

4. **HAProxy → backend**
   ```bash
   nc -vz <cp-ip> 6443
   ```

5. **API health through LB**
   ```bash
   curl -k https://haproxy.faekcorp.lab:6443/livez
   ```

6. **kubeadm certificate upload**
   ```bash
   sudo kubeadm init phase upload-certs --upload-certs
   ```

### kubeadm phase → layer mapping

| Phase shown | Layer being tested |
|---|---|
| `[preflight]` | Networking + basic node checks |
| `[download-certs]` | Certificates via API server |
| `[etcd]` | etcd membership |
| `[kubelet-start]` | kubelet + static pods |

If it hangs at `[preflight]` → networking.
If it reaches `[download-certs]` → networking is fine; problem is kubeadm-level.

---

## 18. Lessons Learned

1. **DNS resolution ≠ TCP connectivity.** Test both separately.
2. **AWS Security Groups are directional.** A TCP proxy requires rules in **both** directions.
3. **`kubeadm init` does not require HAProxy.** The first control plane creates its own API server locally.
4. **`kubeadm join` does require HAProxy.** Joining nodes have no API server yet and must reach the existing cluster through the endpoint.
5. **`kubeadm-certs` is temporary.** Regenerate with `kubeadm init phase upload-certs --upload-certs` when needed.
6. **Read the kubeadm phase output.** It tells you which layer is failing.
7. **Test each side of a proxy independently** before modifying kubeadm.
8. **Do not manually edit kubelet config.** kubeadm creates it during init/join. Only the local-API repoint (`sed`) is deliberate.

---

## 19. Final Architecture

```
                         ┌──────────────────┐
                         │     Bastion      │
                         │     kubectl      │
                         └────────┬─────────┘
                                  │
                                  │ TCP 6443
                                  ▼
                         ┌──────────────────┐
                         │     HAProxy      │
                         │    + BIND9       │
                         │  10.70.11.192    │
                         │     :6443        │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             ▼             ▼
                 cp01          cp02          cp03
              10.70.21.6   10.70.31.209   10.70.41.250
                 :6443         :6443          :6443
                    │             │             │
                    └─────────────┼─────────────┘
                                  │
                            etcd cluster
                            3 members
                            quorum = 2

                 worker1                    worker2
              10.70.61.85                10.70.11.67
```

### AWS Security Group relationships

```
controlplane-sg  ─── TCP 6443 ───►  lb-dns-sg
     cp01                              HAProxy
     cp02                            10.70.11.192
     cp03                                │
                                         │ TCP 6443
                                         ▼
                                   controlplane-sg
                                    cp01/cp02/cp03
```

### kubeadm certificate distribution

```
             cp01
              │
              │ kubeadm init --upload-certs
              ▼
       kubeadm-certs Secret
         (kube-system)
              │
              │ --certificate-key
              ▼
       cp02 / cp03
```

---

## 20. Quick Command Index

| Task | Command |
|---|---|
| Create kubeadm config | `sudo tee /root/kubeadm-ha.yaml <<'EOF' ... EOF` |
| Initialize cp01 | `sudo kubeadm init --config=/root/kubeadm-ha.yaml --upload-certs \| tee /root/kubeadm-init.out` |
| Show join commands | `grep -A2 "kubeadm join" /root/kubeadm-init.out` |
| Repoint kubelet (cp01) | `sudo sed -i "s\|haproxy.faekcorp.lab\|10.70.21.6\|" /etc/kubernetes/kubelet.conf && sudo systemctl restart kubelet` |
| Repoint kubelet (cp02) | `sudo sed -i "s\|haproxy.faekcorp.lab\|10.70.31.209\|" /etc/kubernetes/kubelet.conf && sudo systemctl restart kubelet` |
| Repoint kubelet (cp03) | `sudo sed -i "s\|haproxy.faekcorp.lab\|10.70.41.250\|" /etc/kubernetes/kubelet.conf && sudo systemctl restart kubelet` |
| Copy admin.conf to bastion | `scp /etc/kubernetes/admin.conf ubuntu@bastion:/tmp/admin.conf` |
| Set bastion kubeconfig | `mkdir -p ~/.kube && sudo mv /tmp/admin.conf ~/.kube/config && sudo chown $(id -u):$(id -g) ~/.kube/config && chmod 600 ~/.kube/config` |
| Point bastion at HAProxy | `kubectl config set-cluster kubernetes --server=https://haproxy.faekcorp.lab:6443` |
| Join cp02/cp03 | `sudo kubeadm join haproxy.faekcorp.lab:6443 --token <T> --discovery-token-ca-cert-hash sha256:<H> --control-plane --certificate-key <K>` |
| Join worker | `sudo kubeadm join haproxy.faekcorp.lab:6443 --token <T> --discovery-token-ca-cert-hash sha256:<H>` |
| Regenerate cert key | `sudo kubeadm init phase upload-certs --upload-certs` |
| Regenerate token | `kubeadm token create --print-join-command` |
| List tokens | `kubeadm token list` |
| Get nodes | `kubectl get nodes -o wide` |
| Get control-plane pods | `kubectl get pods -n kube-system -o wide \| grep -E 'NAME\|cp0'` |
| etcd member list | `kubectl -n kube-system exec etcd-cp01 -- etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key member list` |

---
