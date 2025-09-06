# Complete Jenkins CI/CD Setup for Django Project (Beginner's Guide)

This guide provides a complete, step-by-step setup for configuring Jenkins CI/CD for a Django project with Docker, SonarQube, and automated testing.

## 🎯 What We're Building

A complete CI/CD pipeline that automatically:
- Runs tests when code is pushed to GitHub
- Performs code quality analysis with SonarQube
- Builds Docker images
- Pushes images to DockerHub
- (Optional) Deploys to your server

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

## 🌐 Step 4: GitHub Webhook Setup

### 4.1 Install ngrok (for local Jenkins)
```bash
# Install ngrok
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update
sudo apt install ngrok

# Authenticate ngrok (get token from https://ngrok.com/)
ngrok config add-authtoken <YOUR_NGROK_TOKEN>

# Expose Jenkins
ngrok http 8080
```

### 4.2 Configure GitHub Webhook
1. Go to your GitHub repository → Settings → Webhooks → Add webhook
2. Payload URL: `https://<your-ngrok-url>/github-webhook/`
3. Content type: `application/json`
4. Events: Select "Just the push event"
5. Save webhook

---

## 📦 Step 5: SonarQube Project Setup

1. Access SonarQube at `http://your-server-ip:9001`
2. Login with your admin credentials
3. Go to **Projects → Create Project**
4. Choose "Manually"
5. Project key: `django_jobportal` (or your project name)
6. Project name: `Django Job Portal`
7. Use main branch
8. Create project

---

## 🔧 Step 6: Jenkins Pipeline Job Setup

### 6.1 Create Multibranch Pipeline
1. Go to Jenkins Dashboard → New Item
2. Name: `django_jobportal_pipeline`
3. Type: **Multibranch Pipeline**
4. Click OK

### 6.2 Configure Pipeline
**Branch Sources:**
- Add source: GitHub
- Repository HTTPS URL: `https://github.com/yourusername/your-repo.git`
- Credentials: Select your `GIT_CREDENTIALS_ID`

**Build Configuration:**
- Mode: by Jenkinsfile
- Script Path: `Jenkinsfile` (leave as default)

**Scan Multibranch Pipeline Triggers:**
- Check "Periodically if not otherwise run"
- Interval: `1 day`

Save configuration

---

## 📝 Step 7: Create Jenkinsfile

Create a `Jenkinsfile` in your project root:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'your-dockerhub-username/your-repo'
        DOCKER_TAG = "${env.BRANCH_NAME}-${env.GIT_COMMIT.substring(0, 7)}"
    }

    stages {
        stage('Checkout') {
            steps {
                git(
                    url: 'https://github.com/yourusername/your-repo.git',
                    credentialsId: 'GIT_CREDENTIALS_ID',
                    branch: 'main'
                )
            }
        }

        stage('Setup Environment') {
            steps {
                sh 'python -m venv venv'
                sh '. venv/bin/activate && pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                sh '. venv/bin/activate && python manage.py test --noinput'
            }
            post {
                always {
                    junit '**/test-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=django_jobportal \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=$SONAR_HOST_URL \
                    -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${env.DOCKER_IMAGE}:${env.DOCKER_TAG}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'DOCKERHUB_CREDENTIALS_ID') {
                        docker.image("${env.DOCKER_IMAGE}:${env.DOCKER_TAG}").push()
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

---

## ✅ Step 8: Verification

### 8.1 Test Pipeline Manually
1. Go to your Jenkins pipeline
2. Click "Scan Multibranch Pipeline Now"
3. Click "Build Now" on your branch

### 8.2 Verify Steps
- Checkout should succeed
- Tests should run and pass
- SonarQube analysis should complete
- Docker image should build and push

### 8.3 Test Webhook
1. Make a small change to your code
2. Push to GitHub
3. Verify Jenkins automatically starts the pipeline

---

## 🚨 Troubleshooting

### Common Issues and Solutions

**Permission Denied Errors:**
```bash
sudo chmod 666 /var/run/docker.sock
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**SonarQube Not Accessible:**
```bash
# Check SonarQube status
docker ps
docker logs sonarqube

# Fix memory issue
sudo sysctl -w vm.max_map_count=262144
```

**Jenkins Port Not Accessible:**
```bash
sudo ufw allow 8080
sudo ufw allow 9001
sudo ufw enable
```

**Webhook Not Working:**
- Verify ngrok is running
- Check Jenkins URL in Configure System
- Verify webhook URL in GitHub

---

## 📊 Step 9: Monitoring and Improvements

### 9.1 Add Notifications
```groovy
// Add to your Jenkinsfile
post {
    success {
        slackSend(
            channel: '#your-channel',
            message: "Pipeline SUCCESS: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        )
    }
    failure {
        slackSend(
            channel: '#your-channel',
            message: "Pipeline FAILED: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        )
    }
}
```

### 9.2 Add Deployment Stage (Optional)
```groovy
stage('Deploy to Staging') {
    when {
        branch 'main'
    }
    steps {
        sh 'docker-compose down && docker-compose up -d'
    }
}
```

### 9.3 Add Cleanup Stage
```groovy
stage('Cleanup') {
    steps {
        sh 'docker system prune -f'
    }
}
```

---

## 🔄 Complete CI/CD Flow

1. **Developer** pushes code to GitHub
2. **GitHub** sends webhook to Jenkins
3. **Jenkins** automatically starts pipeline
4. **Tests** run and code quality is checked
5. **Docker image** is built and pushed to DockerHub
6. **Notifications** are sent on success/failure
7. **(Optional) Automatic deployment** to staging

---

## 📚 Additional Resources

- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [SonarQube Documentation](https://docs.sonarqube.org/latest/)
- [Docker Documentation](https://docs.docker.com/)
- [Django Deployment Guide](https://docs.djangoproject.com/en/stable/howto/deployment/)

---

## ❓ Getting Help

If you encounter issues:
1. Check Jenkins logs: `sudo journalctl -u jenkins -f`
2. Verify services: `sudo systemctl status jenkins docker`
3. Check SonarQube: `docker logs sonarqube`
4. Consult documentation above

Remember: This setup is for learning. For production, add security hardening, monitoring, and backup procedures.

**Happy Coding! 🚀**
