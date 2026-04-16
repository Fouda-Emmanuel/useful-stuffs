# GitHub Actions Self-Hosted Runner Installation Script

## 📖 What is this?

This doc provides a **bash script** that automatically installs and configures a GitHub Actions self-hosted runner as a **systemd service** on Ubuntu/Linux systems (like EC2 instances).

### Why use this script?

- **Persistent service** - Runner starts automatically on system boot
- **Survives restarts** - Continues working after EC2 stop/start cycles  
- **Production-ready** - Runs as a proper systemd service
- **One-command setup** - Fully automated installation
- **Easy management** - Simple commands to start/stop/check status
- **Auto-checksum validation** - Automatically verifies download integrity

### Use Cases

- **CI/CD on EC2** - Run GitHub Actions workflows on your own AWS infrastructure
- **Custom environments** - Use specific hardware, GPUs, or software configurations
- **Air-gapped setups** - Run runners in private networks or VPCs

---

## 📋 Prerequisites

Before running this script, ensure you have:

- **Ubuntu 20.04+** or similar Linux distribution
- **Root/sudo access** on the target machine
- **GitHub repository** where you'll register the runner
- **Runner token** from your GitHub repository (see below)

---

## 🔧 Installation Script

Copy the script below, replace the **placeholders** (marked with `YOUR_`), and run it on your server.

```bash
#!/bin/bash

# ============================================
# Script: GitHub Actions Runner as Service
# Purpose: Install runner agent and configure as systemd service
# Auto-starts on boot, survives stop/start cycles
# ============================================

set -e  # Stop script on any error

echo "=========================================="
echo "Installing GitHub Actions Runner as Service"
echo "=========================================="

# ============================================
# CONFIGURATION - REPLACE THESE PLACEHOLDERS!
# ============================================

# GitHub repository URL (YOUR repo)
# Example: "https://github.com/your-username/your-repo"
REPO_URL="YOUR_GITHUB_REPO_URL"

# Runner token (Get from GitHub: Settings > Actions > Runners > New self-hosted runner)
# Example: "ABC123DEF456GHI789JKL"
RUNNER_TOKEN="YOUR_RUNNER_TOKEN"

# Runner name (optional, defaults to hostname if left empty)
# Example: "production-runner-01"
RUNNER_NAME="YOUR_RUNNER_NAME"

# Runner labels (comma separated, no spaces)
# Example: "self-hosted,linux,x64,production"
RUNNER_LABELS="YOUR_RUNNER_LABELS"

# Optional: Runner version (check latest at https://github.com/actions/runner/releases)
RUNNER_VERSION="2.333.1"

# ============================================
# DO NOT EDIT BELOW THIS LINE
# ============================================

# 1. Create actions-runner directory
echo "Step 1: Creating actions-runner directory..."
cd /home/ubuntu
mkdir -p actions-runner
cd actions-runner

# 2. Download the latest runner package
echo "Step 2: Downloading GitHub Actions Runner version ${RUNNER_VERSION}..."
curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# 3. Download and verify checksum (automated, no placeholder needed)
echo "Step 3: Downloading and validating checksum..."
curl -o checksums.txt -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz.sha256
shasum -a 256 -c checksums.txt --ignore-missing

# 4. Extract the installer
echo "Step 4: Extracting runner package..."
tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# 5. Configure the runner
echo "Step 5: Configuring runner..."
./config.sh --unattended \
    --url $REPO_URL \
    --token $RUNNER_TOKEN \
    --name $RUNNER_NAME \
    --labels $RUNNER_LABELS \
    --work _work

# 6. Install as systemd service
echo "Step 6: Installing runner as systemd service..."
sudo ./svc.sh install

# 7. Start the service
echo "Step 7: Starting runner service..."
sudo ./svc.sh start

# 8. Check service status
echo "Step 8: Checking service status..."
sudo ./svc.sh status

echo "=========================================="
echo "GitHub Actions Runner Service Installed!"
echo "=========================================="
echo ""
echo "Useful commands:"
echo "  - Check status: sudo ./svc.sh status"
echo "  - Stop runner:  sudo ./svc.sh stop"
echo "  - Start runner: sudo ./svc.sh start"
echo "  - View logs:    sudo ./svc.sh log"
echo ""
echo "To verify runner is registered:"
echo "  Go to GitHub > Settings > Actions > Runners"
echo "=========================================="
```

---

## 🚀 How to Use This Script

### Step 1: Get Your Runner Token

1. Go to your GitHub repository
2. Click **Settings** → **Actions** → **Runners**
3. Click **"New self-hosted runner"**
4. Copy the token from the configuration command (looks like `A1B2C3D4E5F6G7H8I9J0`)

### Step 2: Configure the Script

Replace the following placeholders in the script:

| Placeholder | Description | Example |
|------------|-------------|---------|
| `YOUR_GITHUB_REPO_URL` | Your GitHub repository URL | `https://github.com/octocat/hello-world` |
| `YOUR_RUNNER_TOKEN` | Token from GitHub (step 1) | `A1B2C3D4E5F6G7H8I9J0` |
| `YOUR_RUNNER_NAME` | Unique name for this runner | `production-runner-01` |
| `YOUR_RUNNER_LABELS` | Comma-separated labels | `self-hosted,linux,x64,aws` |

### Step 3: Run the Script

```bash
# Save the script as install-runner.sh
nano install-runner.sh

# Make it executable
chmod +x install-runner.sh

# Run with sudo (required for service installation)
sudo ./install-runner.sh
```

### Step 4: Verify Installation

```bash
# Check service status
cd /home/ubuntu/actions-runner
sudo ./svc.sh status

# You should see: "actions.runner.service - GitHub Actions Runner" (active)
```

---

## 🛠️ Managing Your Runner Service

Once installed, use these commands from `/home/ubuntu/actions-runner`:

```bash
# Check runner status
sudo ./svc.sh status

# Stop the runner
sudo ./svc.sh stop

# Start the runner
sudo ./svc.sh start

# Restart the runner
sudo ./svc.sh restart

# View live logs
sudo ./svc.sh log

# Uninstall the service
sudo ./svc.sh uninstall
```

---

## 🔍 Troubleshooting

### Runner not showing in GitHub
- Wait 30 seconds and refresh
- Check token hasn't expired (tokens expire after 1 hour)
- Verify network connectivity to GitHub

### Service fails to start
```bash
# Check systemd logs
sudo journalctl -u actions.runner.service -f

# Check runner logs
cd /home/ubuntu/actions-runner
sudo ./svc.sh log
```

### Checksum validation fails
```bash
# Skip validation and manually verify
# Or update RUNNER_VERSION to match the latest release
```
### Runner offline after EC2 restart
The service is configured to auto-start. Check if the service is enabled:
```bash
sudo systemctl is-enabled actions.runner.service
```

---

## 📦 Updating the Runner

To update to a newer runner version:

```bash
cd /home/ubuntu/actions-runner
sudo ./svc.sh stop
cd ..
mv actions-runner actions-runner-old
# Update RUNNER_VERSION in the script and re-run
```

---

## 🔐 Security Best Practices

1. **Never commit the script with real tokens** to public repositories
2. **Use GitHub secrets** for tokens in automation
3. **Restrict runner permissions** - Use repository-specific runners for sensitive projects
4. **Regular updates** - Keep runner version updated for security patches
5. **Network isolation** - Run runners in private subnets when possible

---

## ❓ FAQ

**Q: Can I use this on Amazon Linux 2?**  
A: Yes, but change `/home/ubuntu` to `/home/ec2-user`

**Q: How many runners can I register?**  
A: Unlimited - each needs a unique name and token

**Q: Does it work on ARM64?**  
A: Yes, change `x64` to `arm64` in the download URL and filename

**Q: Can I run multiple runners on one machine?**  
A: Yes, create separate directories with different names and ports

**Q: Why does the script auto-fetch checksums?**  
A: So you don't have to manually update hash placeholders - it's fully automated!
