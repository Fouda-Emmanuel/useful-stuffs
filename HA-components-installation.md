### Containerd
#!/bin/bash
set -e

echo "=== 1. Installing repository prerequisites ==="
sudo apt-get update
sudo apt-get install -y ca-certificates curl

echo "=== 2. Installing Docker repository GPG key ==="
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "=== 3. Adding Docker APT repository ==="
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

echo "=== 4. Updating package lists ==="
sudo apt-get update

echo "=== 5. Installing containerd.io 2.2.6 ==="
sudo apt-get install -y \
  containerd.io=2.2.6-1~ubuntu.24.04~noble

echo "=== 6. Preventing automatic containerd upgrade ==="
sudo apt-mark hold containerd.io

echo "=== 7. Generating containerd configuration ==="
sudo mkdir -p /etc/containerd

containerd config default \
  | sudo tee /etc/containerd/config.toml > /dev/null

echo "=== 8. Configuring systemd cgroups ==="
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/g' \
  /etc/containerd/config.toml

echo "=== 9. Configuring Kubernetes sandbox image ==="
sudo sed -i \
  's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.10"#' \
  /etc/containerd/config.toml

echo "=== 10. Starting containerd ==="
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

echo "=== 11. Verification ==="

echo "--- containerd version ---"
containerd --version

echo "--- containerd package ---"
dpkg -l | grep containerd.io

echo "--- systemd cgroup ---"
grep 'SystemdCgroup' /etc/containerd/config.toml

echo "--- sandbox image ---"
grep -i pause /etc/containerd/config.toml

echo "--- service status ---"
systemctl is-active containerd

echo "=== containerd installation complete ==="

-----------------------------------------------------------------------------------------

### Kubeadm, Kubelet

#!/bin/bash
set -e

K8S_VERSION="1.35"
K8S_PACKAGE_VERSION="1.35.8-1.1"

echo "=========================================="
echo " Kubernetes ${K8S_PACKAGE_VERSION}"
echo " Installing kubeadm + kubelet"
echo "=========================================="

# --------------------------------------------------
# 1. Remove any old Kubernetes repository/key
# --------------------------------------------------

sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# --------------------------------------------------
# 2. Prerequisites
# --------------------------------------------------

sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg

# --------------------------------------------------
# 3. Kubernetes repository key
# --------------------------------------------------

sudo mkdir -p -m 0755 /etc/apt/keyrings

curl -fsSL \
  https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# --------------------------------------------------
# 4. Kubernetes v1.35 repository
# --------------------------------------------------

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# --------------------------------------------------
# 5. Install EXACT Kubernetes version
# --------------------------------------------------

sudo apt-get install -y \
  kubelet=${K8S_PACKAGE_VERSION} \
  kubeadm=${K8S_PACKAGE_VERSION}

# --------------------------------------------------
# 6. Prevent automatic package upgrades
# --------------------------------------------------

sudo apt-mark hold kubelet kubeadm

# --------------------------------------------------
# 7. Enable kubelet
#    Do NOT start it manually yet.
# --------------------------------------------------

sudo systemctl enable kubelet

# --------------------------------------------------
# 8. Verification
# --------------------------------------------------

echo
echo "========== Kubernetes versions =========="

kubeadm version
kubelet --version

echo
echo "========== Held packages =========="

apt-mark showhold

echo
echo "========== Kubelet service =========="

systemctl is-enabled kubelet
systemctl is-active kubelet || true

echo
echo "=========================================="
echo " Kubernetes installation completed."
echo " kubelet is ENABLED but intentionally NOT"
echo " started manually before kubeadm init/join."
echo "=========================================="
