# Terraform & AWS CLI Setup Guide for Ubuntu

A comprehensive guide to set up Terraform, AWS CLI, and VS Code extensions for infrastructure as code development on Ubuntu. Perfect for developers starting with Terraform or setting up a new development environment.

## Prerequisites

- Ubuntu 18.04 or later
- sudo privileges
- Internet connection

**Check your CPU architecture:**
```bash
uname -m
```
- **x86_64** → Intel/AMD CPU (use standard installation)
- **aarch64/arm64** → ARM CPU (use ARM-specific installation)

---

## 1. Install Terraform

### Method 1: Using Official HashiCorp Repository (Recommended)

```bash
# Update package index and install prerequisites
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# Install HashiCorp's GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

# Verify the GPG key's fingerprint
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint
```

**Expected fingerprint output:**
```
/usr/share/keyrings/hashicorp-archive-keyring.gpg
-------------------------------------------------
pub   rsa4096 XXXX-XX-XX [SC]
      XXXX XXXX XXXX XXXX XXXX  XXXX XXXX XXXX XXXX XXXX
uid           [ unknown] HashiCorp Security (HashiCorp Package Signing) <security+packaging@hashicorp.com>
sub   rsa4096 XXXX-XX-XX [E]
```

```bash
# Add the official HashiCorp repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update package list and install Terraform
sudo apt update
sudo apt-get install terraform
```

### Method 2: Manual installation (Latest version)

```bash
# Download latest Terraform version
TERRAFORM_VERSION=$(curl -s https://api.github.com/repos/hashicorp/terraform/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//')
wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip

# Install unzip if not available
sudo apt install -y unzip

# Extract and install
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Clean up
rm terraform_${TERRAFORM_VERSION}_linux_amd64.zip
```

### Verify the Installation

```bash
# Verify installation worked
terraform -help

# Check version
terraform --version
```

**Expected help output:**
```
Usage: terraform [global options] <subcommand> [args]

The available commands for execution are listed below.
The primary workflow commands are given first, followed by
less common or more advanced commands.

Main commands:
  init          Prepare your working directory for other commands
  validate      Check whether the configuration is valid
  plan          Show changes required by the current configuration
  apply         Create or update infrastructure
  destroy       Destroy previously-created infrastructure
```

---

## 2. Install and Configure AWS CLI

### Check Your CPU Architecture
```bash
uname -m
```
- **If output is `x86_64`** → Use standard installation (below)
- **If output is `aarch64` or `arm64`** → Use ARM installation (see ARM section)

### Install AWS CLI v2 (for Intel/AMD x86_64)

```bash
# Download AWS CLI v2 for x86_64
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Install unzip if not available
sudo apt install -y unzip

# Extract and install
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Clean up
rm -rf awscliv2.zip aws/
```

### Install AWS CLI v2 (for ARM CPUs - aarch64/arm64 only)

```bash
# Download AWS CLI v2 for ARM
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"

# Install unzip if not available
sudo apt install -y unzip

# Extract and install
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Clean up
rm -rf awscliv2.zip aws/
```

### Alternative: Quick install with verification

```bash
# For x86_64 systems
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
sudo apt install unzip -y && \
unzip awscliv2.zip && \
sudo ./aws/install && \
rm -rf awscliv2.zip aws/ && \
aws --version

# For ARM systems
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip" && \
sudo apt install unzip -y && \
unzip awscliv2.zip && \
sudo ./aws/install && \
rm -rf awscliv2.zip aws/ && \
aws --version
```

### Configure AWS CLI

```bash
# Configure AWS credentials
aws configure
```

You'll be prompted to enter:
- **AWS Access Key ID**: Your AWS access key
- **AWS Secret Access Key**: Your AWS secret key
- **Default region name**: (e.g., `us-east-1`, `eu-west-1`)
- **Default output format**: (e.g., `json`, `yaml`, `text`)

**Alternative: Using environment variables**

```bash
# Add to ~/.bashrc or ~/.zshrc
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"

# Reload your shell configuration
source ~/.bashrc
```

### Verify AWS CLI Configuration

```bash
# Test AWS CLI configuration
aws sts get-caller-identity

# List S3 buckets (if you have any)
aws s3 ls
```

---

## 3. Install VS Code and Extensions

### Install VS Code

```bash
# Method 1: Using snap (Recommended)
sudo snap install --classic code

# Method 2: Using official Microsoft repository
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c echo 'deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main' > /etc/apt/sources.list.d/vscode.list

sudo apt update
sudo apt install -y code

# Clean up
rm packages.microsoft.gpg
```

### Essential VS Code Extensions for Terraform

Install these extensions via VS Code Extensions marketplace (Ctrl+Shift+X):

```bash
# Install extensions via command line (optional)
code --install-extension hashicorp.terraform
code --install-extension ms-azuretools.vscode-azureterraform
code --install-extension redhat.vscode-yaml
code --install-extension eamodio.gitlens
code --install-extension ms-vscode.makefile-tools
code --install-extension shardulm94.trailing-spaces
```

**Recommended Extensions:**
1. **HashiCorp Terraform** - Syntax highlighting, validation, and formatting
2. **Azure Terraform** - Additional Terraform support
3. **YAML** - YAML language support
4. **GitLens** - Git supercharged
5. **Trailing Spaces** - Highlight trailing spaces

### Configure VS Code for Terraform

Create VS Code settings file:

```bash
# Create .vscode directory in your project
mkdir -p ~/terraform-projects/.vscode

# Create settings.json
cat > ~/terraform-projects/.vscode/settings.json << EOF
{
    "terraform.languageServer": {
        "enabled": true,
        "args": ["serve"]
    },
    "files.associations": {
        "*.tf": "terraform",
        "*.tfvars": "terraform"
    },
    "editor.formatOnSave": true,
    "terraform.format": {
        "ignoreExtensionsOnSave": [
            ".tftpl",
            ".tfvars"
        ]
    },
    "[terraform]": {
        "editor.defaultFormatter": "hashicorp.terraform",
        "editor.formatOnSave": true
    }
}
EOF
```

---

## 4. Verify Complete Setup

### Create a Test Terraform Project

```bash
# Create project directory
mkdir ~/terraform-test
cd ~/terraform-test

# Create main.tf file
cat > main.tf << EOF
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

output "configuration_check" {
  value = "Terraform and AWS CLI are properly configured!"
}
EOF

# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# Plan (this will test AWS credentials)
terraform plan
```

### Expected Output
You should see:
- ✅ Terraform initialization successful
- ✅ Configuration validation passed
- ✅ AWS authentication successful
- ✅ Terraform plan generated without errors

---

## 5. Quick Setup Script

Create a setup script for future use:

```bash
# create setup script with official HashiCorp method
cat > terraform-setup.sh << 'EOF'
#!/bin/bash

echo "Starting Terraform and AWS CLI setup using official methods..."
echo "Detecting system architecture..."

# Check architecture
ARCH=$(uname -m)
echo "Architecture: $ARCH"

# Update system and install prerequisites for Terraform
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# Install HashiCorp's GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "Verifying GPG key fingerprint..."
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint

# Add HashiCorp repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install Terraform
sudo apt update
sudo apt-get install terraform -y

# Install AWS CLI based on architecture
if [ "$ARCH" = "x86_64" ]; then
    echo "Installing AWS CLI for x86_64..."
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
elif [ "$ARCH" = "aarch64" ] || [ "$ARCH" = "arm64" ]; then
    echo "Installing AWS CLI for ARM..."
    curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
else
    echo "Unsupported architecture: $ARCH"
    exit 1
fi

sudo apt install -y unzip
unzip awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws/

# Install VS Code
sudo snap install --classic code

echo "Setup complete!"
echo "Please configure AWS CLI using: aws configure"
echo "Install VS Code extensions manually from marketplace"
echo "Verify installation with: terraform -help && aws --version"
EOF

# Make script executable
chmod +x terraform-setup.sh
```

---

## 6. Troubleshooting

### Common Issues and Solutions

**Terraform not found:**
```bash
export PATH=$PATH:/usr/local/bin
```

**Terraform GPG key verification warning:**
```bash
# This warning is normal and expected:
# "WARNING: This key is not certified with a trusted signature!"
# It occurs because there isn't a chain of trust between your PGP key and HashiCorp's key
```

**AWS CLI authentication errors:**
```bash
# Check credentials
aws configure list

# Test with simple command
aws sts get-caller-identity
```

**Terraform init fails:**
```bash
# Clean and reinitialize
rm -rf .terraform
terraform init
```

**Wrong AWS CLI architecture:**
```bash
# Check current AWS CLI info
aws --version
which aws

# Reinstall with correct architecture
# Follow the installation steps above for your CPU type
```

**VS Code Terraform extension not working:**
- Reload VS Code window (Ctrl+Shift+P → "Developer: Reload Window")
- Ensure the Terraform binary is in your PATH

---

## 7. Maintenance

### Update Terraform
```bash
sudo apt update && sudo apt upgrade terraform
```

### Update AWS CLI
```bash
# For both x86_64 and ARM
sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
```

### Useful Aliases
Add to your `~/.bashrc` or `~/.zshrc`:
```bash
# Terraform aliases
alias tf='terraform'
alias tfi='terraform init'
alias tfp='terraform plan'
alias tfa='terraform apply'
alias tfd='terraform destroy'
alias tfv='terraform validate'
alias tff='terraform fmt'

# AWS aliases
alias aws-whoami='aws sts get-caller-identity'
alias aws-region='aws configure get region'

# System info
alias myarch='uname -m'
```

---

## 🎉 Setup Complete!

Your development environment is now ready for Terraform projects. You have:

- ✅ Terraform installed and configured using official HashiCorp method
- ✅ AWS CLI set up with correct architecture
- ✅ VS Code with essential extensions
- ✅ Test project verified and working

### Next Steps:
1. Start creating your Terraform configurations
2. Version control your code with Git
3. Explore Terraform modules and best practices
4. Set up remote state storage (S3 + DynamoDB)

---

## 🚀 Quick Commands Reference

### System Information
```bash
uname -m                    # Check CPU architecture
lsb_release -a             # Check Ubuntu version
```

### Installation Commands
```bash
# Quick Terraform install (Official method)
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common && \
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list && \
sudo apt update && sudo apt-get install terraform -y

# Quick AWS CLI install (x86_64)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
sudo apt install unzip -y && unzip awscliv2.zip && sudo ./aws/install && rm awscliv2.zip

# Quick VS Code install
sudo snap install --classic code
```

### Daily Terraform Commands
```bash
terraform init          # Initialize project
terraform plan          # Preview changes
terraform apply         # Apply changes
terraform destroy       # Destroy infrastructure
terraform fmt           # Format code
terraform validate      # Validate configuration
terraform show          # Show current state
terraform output        # Show outputs
```

### AWS CLI Commands
```bash
aws configure                    # Configure AWS credentials
aws sts get-caller-identity     # Verify credentials
aws s3 ls                       # List S3 buckets
aws configure list              # Show current configuration
```

### Verification Commands
```bash
# Verify complete setup
terraform -help && aws --version && code --version

# Verify AWS configuration
aws sts get-caller-identity

# Verify architecture compatibility
uname -m && aws --version
```

### One-Liner Complete Setup (x86_64)
```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl unzip && \
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list && \
sudo apt update && sudo apt-get install terraform -y && \
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install && rm awscliv2.zip && \
sudo snap install --classic code && \
echo "Setup complete! Run 'aws configure' to set up credentials."
```

### Quick Test
```bash
# Quick test of entire setup
mkdir -p /tmp/terraform-test && cd /tmp/terraform-test && \
echo 'terraform { required_providers { aws = { source = "hashicorp/aws" } } }' > main.tf && \
terraform init && terraform validate && \
aws sts get-caller-identity && \
echo "✅ All systems go!"
```

**Happy Terraforming! 🚀**

---

*This documentation follows official HashiCorp installation methods. For updates or issues, please check the GitHub repository or create an issue.*
