# Jenkins CI/CD Setup for Django Project 

This comprehensive guide will walk you through setting up Jenkins for a Django project with Docker, SonarQube, and automated testing. Beginner friendly!

## 🎯 High-Level Goal

When code is pushed to your repository, Jenkins will automatically:
1. Pull the latest code
2. Build Docker containers
3. Run tests and collect coverage reports
4. Perform SonarQube analysis with quality gate checks
5. Push Docker images to DockerHub (if all tests pass)
6. Optionally deploy to your server
7. Send notifications (optional)

---

## 📋 Prerequisites

Before starting, ensure you have:
- A server/VM with Ubuntu 20.04+ (or similar Linux distribution)
- Basic terminal/command line knowledge
- A Django project in a Git repository (GitHub, GitLab, or Bitbucket)
- Administrative access to install software

---

## 🚀 Step 1: Server Preparation

### 1.1 Update System Packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip
```

### 1.2 Install Java (Required for Jenkins)
```bash
sudo apt install -y openjdk-17-jdk
java -version  # Verify installation
```

### 1.3 Install Docker
```bash
# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER  # Add current user to docker group

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### 1.4 Install Jenkins
```bash
# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install -y jenkins

# Start Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Check status
sudo systemctl status jenkins
```

### 1.5 Initial Jenkins Setup
1. Open your browser and go to `http://your-server-ip:8080`
2. Unlock Jenkins using the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Install suggested plugins
4. Create admin user account
5. Jenkins is now ready!

---

## 🔐 Step 2: Generate Required Tokens

### 2.1 GitHub/GitLab Personal Access Token
1. Go to your Git provider (GitHub/GitLab) → Settings → Developer Settings
2. Generate a new token with these permissions:
   - `repo` (full access to repositories)
   - `read:user` (read user profile data)
   - `admin:repo_hook` (manage repository webhooks)
3. Copy and save the token securely

### 2.2 DockerHub Access Token
1. Log in to DockerHub → Account Settings → Security → Access Tokens
2. Generate a new token with:
   - Read & Write permissions
   - Description: "Jenkins CI/CD"
3. Copy and save the token securely

### 2.3 SonarQube Token
1. Start SonarQube (if not already running):
   ```bash
   docker run -d --name sonarqube -p 9001:9000 sonarqube:lts-community
   ```
2. Wait a few minutes, then access `http://your-server-ip:9001`
3. Default credentials: admin/admin (you'll be prompted to change password)
4. Go to My Account → Security → Generate Tokens
5. Name: "Jenkins-Scanner"
6. Copy and save the token securely

---

## 🛠️ Step 3: Jenkins Configuration

### 3.1 Install Required Plugins
Go to **Manage Jenkins → Manage Plugins → Available** and install:
- Git
- Pipeline
- Docker Pipeline
- SonarQube Scanner
- JUnit
- Credentials Binding
- Blue Ocean (optional, for better UI)

### 3.2 Add Credentials to Jenkins
Go to **Jenkins → Credentials → System → Global credentials → Add Credentials**

**Git Credentials:**
- Kind: Secret text
- Secret: [Your Git PAT token]
- ID: `GIT_CREDENTIALS_ID`
- Description: Git Personal Access Token

**DockerHub Credentials:**
- Kind: Username with password
- Username: [Your DockerHub username]
- Password: [Your DockerHub access token]
- ID: `DOCKERHUB_CREDENTIALS_ID`
- Description: DockerHub Access Token

**SonarQube Token:**
- Kind: Secret text
- Secret: [Your SonarQube token]
- ID: `SONAR_TOKEN_ID`
- Description: SonarQube Analysis Token

### 3.3 Configure Global Tools
Go to **Manage Jenkins → Global Tool Configuration**

**JDK:**
- Name: `JDK17`
- JAVA_HOME: `/usr/lib/jvm/java-17-openjdk-amd64`

**Git:**
- Name: `Default`
- Path to Git executable: `git`

**SonarQube Scanner:**
- Name: `sonar-scanner`
- Check "Install automatically"
- Version: `7.2.0.5079`

### 3.4 Configure SonarQube Server
Go to **Manage Jenkins → Configure System → SonarQube servers**

- Click "Add SonarQube"
- Name: `sonarqube-server`
- Server URL: `http://localhost:9001` (or your SonarQube URL)
- Server authentication token: Select your `SONAR_TOKEN_ID` from dropdown
- Check "Environment variables" (IMPORTANT!)

---

## 📝 Step 4: Pipeline Planning

Before creating your Jenkinsfile, understand the workflow:

1. **Checkout**: Clone repository using Git credentials
2. **Setup**: Prepare environment and dependencies
3. **Test**: Run Django tests with coverage
4. **SonarQube Analysis**: Code quality checks
5. **Build**: Create Docker image
6. **Push**: Upload to DockerHub (if tests pass)
7. **Deploy**: Optional deployment to server
8. **Notify**: Send success/failure notifications

---

## ✅ Verification Steps

### Test Jenkins → SonarQube Connection
1. Create a new Pipeline job
2. Use this test script:
```groovy
pipeline {
    agent any
    stages {
        stage('Test SonarQube Connection') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh 'echo "Connected to $SONAR_HOST_URL"'
                }
            }
        }
    }
}
```

### Test Git Integration
1. Create a new Pipeline job
2. Configure Git repository URL
3. Select your Git credentials from dropdown
4. Run job to verify clone works

---

## 🚨 Troubleshooting Common Issues

### Permission Denied Errors
```bash
# Fix Docker permissions
sudo chmod 666 /var/run/docker.sock

# Fix Jenkins permissions
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### SonarQube Not Accessible
```bash
# Check SonarQube status
docker ps
docker logs sonarqube

# If SonarQube won't start (common memory issue)
sudo sysctl -w vm.max_map_count=262144
```

### Jenkins Port Not Accessible
```bash
# Check firewall
sudo ufw allow 8080
sudo ufw allow 9001
sudo ufw enable
```

---

## 📚 Next Steps

After completing this setup:
1. Create your `Jenkinsfile` in your Django project repository
2. Configure webhooks in your Git provider to trigger Jenkins on push
3. Set up environment-specific configurations (dev/staging/prod)
4. Add monitoring and alerting
5. Implement backup strategies for Jenkins and SonarQube data

---

## 🔗 Useful Resources

- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [SonarQube Documentation](https://docs.sonarqube.org/latest/)
- [Docker Documentation](https://docs.docker.com/)
- [Django Deployment Checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)

---

## ❓ Getting Help

If you encounter issues:
1. Check Jenkins logs: `sudo journalctl -u jenkins -f`
2. Verify all services are running: `sudo systemctl status jenkins docker`
3. Check SonarQube logs: `docker logs sonarqube`
4. Consult the Jenkins and SonarQube documentation

Remember: This setup is for learning purposes. For production environments, add security hardening, monitoring, and backup procedures.

**Happy Coding! 🚀**
