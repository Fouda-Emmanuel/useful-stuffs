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
