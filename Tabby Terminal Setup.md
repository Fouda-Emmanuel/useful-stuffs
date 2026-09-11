# Tabby Terminal Setup Guide 

This guide documents how to install **Tabby Terminal** on Ubuntu and configure a rock-solid, multi-tab SSH workspace with a ** Bastion Jump Host (ProxyJump)** routing system for managing an internal Kubernetes cluster.

---

## 1. Installation (Verified on Ubuntu 24.04 LTS "Noble")

The most reliable way to install Tabby Terminal and keep it updated natively is by using its official Packagecloud repository. Run this single block in your terminal:

```bash
# Add the official Tabby repository and signing keys
curl -s https://packagecloud.io/install/repositories/eugeny/tabby/script.deb.sh | sudo bash

# Update your system package index and install Tabby
sudo apt update && sudo apt install -y tabby-terminal
```

---

## 2. Permanent SSH Keepalive Configuration (No More Disappearing Tabs!)

To prevent cluster nodes or cloud service firewalls from silently dropping connections when you leave tabs idle, Tabby must be configured with a stable background heartbeat.

When creating or editing **any profile**, apply these values under the connection layout settings:

* **Keep Alive Interval (Milliseconds):** `30000` *(Sends a background micro-ping every 30 seconds)*
* **Max Keep Alive Count:** `240` *(Allows tabs to survive up to 2 hours of heavy server lag or network hiccups before dropping)*
* **Ready Timeout (Milliseconds):** `20000`

---

## 3. Cluster Connection Architecture (Free Jump Host Setup)

Instead of complex scripting, Tabby allows you to link internal private network nodes straight through an external edge node completely for free. 

### Step A: The Bastion Gateway Profile
Create this profile first so it can act as your gateway bridge.
1. Go to **Settings (Gear Icon)** -> **Profiles & connections** -> **New Profile** -> **SSH connection**.
2. **Name:** `Bastion`
3. **Host:** `<YOUR_PUBLIC_BASTION_IP>` *(e.g., 192.0.2.10)*
4. **Username:** `<YOUR_SSH_USER>` *(e.g., ubuntu)*
5. **Private Keys:** Provide the absolute path to your local identity key file (e.g., `file:///home/<your_user>/.ssh/<your_key>.pem`)
6. Adjust the **Keep Alive Settings** (from Section 2) and click **Save**.

### Step B: The Internal Cluster Nodes (Controlplane & Workers)
Repeat this step for your master nodes and worker agents to map them under a unified interface group.
1. Click **New Profile** -> **SSH connection**.
2. **Name:** `Controlplane` *(or `Worker1`, `Worker2`, etc.)*
3. **Group:** `k8s-cluster` *(This clusters them neatly into a dropdown layout in your sidebar menu)*
4. Go to the **Connection** tab:
   * **Host:** `<YOUR_INTERNAL_PRIVATE_IP>` *(e.g., 10.0.0.18, 10.0.0.250, etc.)*
   * **Username:** `<YOUR_CLUSTER_USER>`
   * **Private Keys:** Reference your matching cluster key file path.
5. Go to the **Advanced** tab at the bottom:
   * Locate the **Jump Host** setting field dropdown.
   * Select your saved **`Bastion`** gateway profile!
6. Click **Save**.

---

## 4. Workspace Features to Remember

* **Drag-and-Drop Tabs:** Click and hold any open terminal tab to slide it left and right freely. This lets you order your controlplane and workers sequentially on your monitor.
* **Session Multiplexing:** Toggle the **Reuse session for multiple tabs** feature inside profile properties to significantly reduce connection handshake delays when opening multi-pane cluster tasks.
