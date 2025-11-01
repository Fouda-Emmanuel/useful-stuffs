
# SSL Generation and Deployment Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Infrastructure Setup](#infrastructure-setup)
3. [SSL Certificate Generation](#ssl-certificate-generation)
4. [Portainer Installation & Issues](#portainer-installation--issues)
5. [Security Group Configuration](#security-group-configuration)
6. [Docker Compose Architecture](#docker-compose-architecture)
7. [Nginx Configuration](#nginx-configuration)
8. [Deployment Script](#deployment-script)
9. [Lessons Learned](#lessons-learned)
10. [SSL Certificate Auto-renewal Guide](#ssl-certificate-auto-renewal-guide)

## Project Overview

Foudawire is a Django-based application with a microservices architecture deployed using Docker Compose. The stack includes:
- Django API
- PostgreSQL database
- Redis for caching
- RabbitMQ for message queue
- Celery workers and beat for background tasks
- Flower for Celery monitoring
- Nginx as reverse proxy
- Prometheus & Grafana for monitoring
- Node-exporter for system metrics

## Infrastructure Setup

### EC2 Instance Creation
- Created AWS EC2 instance (Ubuntu 22.04 LTS)
- Instance type: Based on project requirements
- Key pair: Generated and downloaded for SSH access

### Domain Configuration
- Purchased domain: `foudadev.site`
- Configured DNS A records pointing to EC2 public IP:
  - `foudadev.site` → EC2_IP
  - `api.foudadev.site` → EC2_IP
  - `flower.foudadev.site` → EC2_IP
  - `portainer.foudadev.site` → EC2_IP

### Docker Installation
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install docker-compose-plugin -y
```

## SSL Certificate Generation

### Prerequisites
Before generating SSL certificates, ensure:
- Domain DNS records are propagated
- Port 80 is open in security groups
- No services are blocking port 80

### Certificate Generation Process

1. **Create necessary directories:**
```bash
sudo mkdir -p /var/lib/letsencrypt
sudo mkdir -p /etc/letsencrypt
```

2. **Generate certificates using Certbot in standalone mode:**
```bash
sudo docker run -it --rm \
  -p 80:80 \
  -v "/etc/letsencrypt:/etc/letsencrypt" \
  -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
  certbot/certbot certonly --standalone \
  -d foudadev.site \
  -d api.foudadev.site \
  -d flower.foudadev.site \
  -d portainer.foudadev.site \
  --email leoemmanuelson46@gmail.com --agree-tos
```

3. **Certificate output:**
```
Certificate is saved at: /etc/letsencrypt/live/foudadev.site/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/foudadev.site/privkey.pem
Expires: 2026-01-29
```

### Important Notes:
- Certbot runs in standalone mode, requiring port 80 to be free
- Certificates are stored in `/etc/letsencrypt/live/foudadev.site/`
- Auto-renewal setup recommended for production

## Portainer Installation & Issues

### Initial Installation
```bash
# Create volume for Portainer data persistence
docker volume create portainer_data

# Run Portainer container
docker run -d -p 8000:8000 -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

### Issues Encountered

1. **Connection Refused Error**
   - **Problem**: `https://13.218.36.173:9443/` refused connection
   - **Cause**: Security group didn't allow ports 8000 and 9443
   - **Solution**: Added custom TCP rules for ports 8000 and 9443

2. **Portainer Timeout Security Issue**
   - **Problem**: "Your Portainer instance timed out for security purposes"
   - **Cause**: Initial setup timeout security feature
   - **Solution**: Restart Portainer container
   ```bash
   docker restart portainer
   ```

### Final Working Portainer Configuration
```bash
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

## Security Group Configuration

### Initial Setup (Insufficient)
```yaml
- HTTP (80): 0.0.0.0/0
- SSH (22): 129.0.189.27/32
- HTTPS (443): 0.0.0.0/0
- HTTP IPv6 (80): ::/0
- HTTPS IPv6 (443): ::/0
```

### Required Additions for Full Functionality
```yaml
- Custom TCP (8000): 0.0.0.0/0    # Portainer HTTP
- Custom TCP (9443): 0.0.0.0/0    # Portainer HTTPS
- Custom TCP (5432): 0.0.0.0/0    # PostgreSQL (if external access needed)
- Custom TCP (5555): 0.0.0.0/0    # Flower
- Custom TCP (3000): 0.0.0.0/0    # Grafana
- Custom TCP (9090): 0.0.0.0/0    # Prometheus
```

## Docker Compose Architecture

### Key Services Configuration

```yaml
services:
  api: &api
    build: .
    volumes:
      - static_volume:/app/staticfiles
      - app_logs:/app/logs
    expose: ["8000"]
    env_file: ./.envs/.env.production
    depends_on:
      - postgres
      - redis
      - rabbitmq

  nginx:
    build: ./docker/production/nginx
    volumes:
      - ./docker/production/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - static_volume:/app/staticfiles
    ports:
      - '80:80'
      - '443:443'
```

### Network Configuration
```yaml
networks:
  foudawire_prod_nw:
    external: true
```

**Note**: External network must be created manually:
```bash
docker network create foudawire_prod_nw
```

## Nginx Configuration

### Key Features
- HTTP to HTTPS redirect
- SSL termination with modern protocols
- Multiple server blocks for different subdomains
- Static file serving with caching
- Reverse proxy to backend services

### Critical SSL Configuration
```nginx
ssl_certificate /etc/letsencrypt/live/foudadev.site/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/foudadev.site/privkey.pem;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256...;
```

## Deployment Script

### Script Overview
```bash
#!/bin/bash
set -e

# Environment validation
if [ -z "$DIGITAL_OCEAN_IP_ADDRESS" ]; then
  echo "Error: DIGITAL_OCEAN_IP_ADDRESS not defined"
  exit 1
fi

# Archive and transfer project
git archive --format tar --output ./project.tar main
rsync -avz --progress ./project.tar root@$DIGITAL_OCEAN_IP_ADDRESS:/tmp/project.tar

# Remote execution
ssh root@$DIGITAL_OCEAN_IP_ADDRESS <<'ENDSSH'
  # Extract and deploy
  TEMP_DIR=$(mktemp -d)
  tar -xf /tmp/project.tar -C "$TEMP_DIR"
  
  # Stop existing services
  docker compose -f "$TEMP_DIR/production.yml" down --remove-orphans
  
  # Cleanup and rebuild
  docker system prune -af
  docker compose -f "$TEMP_DIR/production.yml" up --build -d --remove-orphans
ENDSSH
```

## Lessons Learned

### 1. SSL Certificate Generation
- **Challenge**: Certbot requires port 80 to be free during certificate generation
- **Solution**: Use standalone mode and ensure no services are running on port 80 during certificate creation
- **Best Practice**: Schedule certificate renewal during maintenance windows

### 2. Security Group Configuration
- **Challenge**: Forgot to open application-specific ports (9443, 8000)
- **Solution**: Systematic port auditing before deployment
- **Checklist**:
  - Web ports (80, 443)
  - Application ports (8000, 9443, 5555, etc.)
  - Monitoring ports (3000, 9090, 9100)
  - Database ports (5432) - restrict to internal only

### 3. Portainer Issues
- **Challenge**: Security timeout on first installation
- **Solution**: Documented restart procedure for security timeouts
- **Insight**: Portainer has built-in security features that require initial configuration

### 4. Docker Network Management
- **Challenge**: External networks must be pre-created
- **Solution**: Add network creation to deployment checklist
- **Command**: `docker network create foudawire_prod_nw`

### 5. Deployment Strategy
- **Success**: Zero-downtime deployment with proper container orchestration
- **Improvement**: Add health checks and rolling updates
- **Backup**: Always backup databases before deployment

## Future Improvements

1. **SSL Certificate Auto-renewal**
   ```bash
   # Add to crontab
   0 12 * * * docker run --rm -v /etc/letsencrypt:/etc/letsencrypt -v /var/lib/letsencrypt:/var/lib/letsencrypt certbot/certbot renew && docker compose restart nginx
   ```

2. **Monitoring & Alerting**
   - Set up Prometheus alerts
   - Configure Grafana dashboards
   - Monitor certificate expiration

3. **Security Hardening**
   - Implement fail2ban
   - Regular security updates
   - Database access restrictions

4. **Backup Strategy**
   - Automated database backups
   - Configuration backups
   - Disaster recovery procedures
---

# SSL Certificate Auto-renewal Guide

## Table of Contents
1. [Quick Start (Simplest Method)](#quick-start-simplest-method)
2. [Method 1: Basic Crontab (Recommended for Beginners)](#method-1-basic-crontab-recommended-for-beginners)
3. [Method 2: Script-Based (Recommended for Production)](#method-2-script-based-recommended-for-production)
4. [Method 3: Docker Compose Integrated (Advanced)](#method-3-docker-compose-integrated-advanced)
5. [Verification & Testing](#verification--testing)
6. [Troubleshooting](#troubleshooting)

---

## Quick Start (Simplest Method)

**Use this if you want the fastest setup:**

```bash
# Add to crontab (run every Monday at 3 AM)
sudo crontab -e

# Add this line:
0 3 * * 1 docker compose -f /app/production.yml stop nginx && docker run --rm -v "/etc/letsencrypt:/etc/letsencrypt" -v "/var/lib/letsencrypt:/var/lib/letsencrypt" -p 80:80 certbot/certbot renew && docker compose -f /app/production.yml start nginx
```

**That's it!** Your certificates will auto-renew.

---

## Method 1: Basic Crontab (Recommended for Beginners)

### Step-by-Step Setup

#### Step 1: Edit Crontab
```bash
sudo crontab -e
```

#### Step 2: Add Auto-renewal Entry
Choose one schedule:

**Option A: Weekly (Recommended)**
```bash
# Run every Monday at 3 AM
0 3 * * 1 docker compose -f /app/production.yml stop nginx && docker run --rm -v "/etc/letsencrypt:/etc/letsencrypt" -v "/var/lib/letsencrypt:/var/lib/letsencrypt" -p 80:80 certbot/certbot renew && docker compose -f /app/production.yml start nginx
```

**Option B: Twice Monthly**
```bash
# Run on 1st and 15th at 2:30 AM
30 2 1,15 * * docker compose -f /app/production.yml stop nginx && docker run --rm -v "/etc/letsencrypt:/etc/letsencrypt" -v "/var/lib/letsencrypt:/var/lib/letsencrypt" -p 80:80 certbot/certbot renew && docker compose -f /app/production.yml start nginx
```

#### Step 3: Save and Exit
- Press `Ctrl + X`
- Press `Y` to save
- Press `Enter` to confirm

### Verification
```bash
# Check if crontab was added
sudo crontab -l

# Check cron logs (after it should have run)
grep CRON /var/log/syslog
```

**Pros:**
- Simple, one-time setup
- No additional files needed
- Easy to modify

**Cons:**
- No logging by default
- Harder to debug if issues occur

---

## Method 2: Script-Based (Recommended for Production)

### Complete Setup

#### Step 1: Create Renewal Script
```bash
sudo nano /root/renew-ssl.sh
```

#### Step 2: Add Script Content
```bash
#!/bin/bash

# SSL Auto-renewal Script
echo "$(date): Starting SSL renewal..."

# Stop nginx to free port 80
echo "Stopping nginx..."
docker compose -f /app/production.yml stop nginx

# Renew certificates
echo "Renewing certificates..."
docker run --rm \
  -v "/etc/letsencrypt:/etc/letsencrypt" \
  -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
  -p 80:80 \
  certbot/certbot renew

# Restart nginx
echo "Restarting nginx..."
docker compose -f /app/production.yml start nginx

echo "$(date): SSL renewal completed"
```

#### Step 3: Make Script Executable
```bash
sudo chmod +x /root/renew-ssl.sh
```

#### Step 4: Set up Crontab with Logging
```bash
sudo crontab -e
```

Add:
```bash
# Run every Monday at 3 AM with logging
0 3 * * 1 /root/renew-ssl.sh >> /var/log/ssl-renewal.log 2>&1
```

#### Step 5: Create Log Rotation
```bash
sudo nano /etc/logrotate.d/ssl-renewal
```

Add:
```bash
/var/log/ssl-renewal.log {
    monthly
    rotate 6
    compress
    missingok
    notifempty
}
```

### Usage
```bash
# Test script manually
sudo /root/renew-ssl.sh

# Check logs
tail -f /var/log/ssl-renewal.log
```

**Pros:**
- Better logging
- Easier to modify and maintain
- Production-ready
- Log rotation prevents disk space issues

**Cons:**
- More setup required
- Multiple files to manage

---

## Method 3: Docker Compose Integrated (Advanced)

### Complete Integrated Solution

#### Step 1: Create Advanced Renewal Script
```bash
sudo nano /root/ssl-renewal-advanced.sh
```

```bash
#!/bin/bash

# Advanced SSL Renewal with Error Handling
RENEWAL_LOG="/var/log/ssl-renewal-advanced.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> $RENEWAL_LOG
}

send_alert() {
    # Add your notification method here (email, Slack, etc.)
    log "ALERT: $1"
    # Example: echo "$1" | mail -s "SSL Renewal Alert" admin@foudadev.site
}

check_ssl_status() {
    log "Checking current SSL status..."
    docker run --rm \
        -v "/etc/letsencrypt:/etc/letsencrypt" \
        -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
        certbot/certbot certificates >> $RENEWAL_LOG 2>&1
}

renew_ssl() {
    log "=== Starting SSL renewal process ==="
    
    # Pre-check: Verify nginx is running
    if ! docker ps | grep -q nginx; then
        log "WARNING: Nginx is not running"
    fi
    
    # Stop nginx
    log "Stopping nginx container..."
    docker compose -f /app/production.yml stop nginx
    
    # Renew certificates
    log "Attempting certificate renewal..."
    if docker run --rm \
        -v "/etc/letsencrypt:/etc/letsencrypt" \
        -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
        -p 80:80 \
        certbot/certbot renew \
        --non-interactive \
        --standalone; then
        
        log "SUCCESS: Certificates renewed successfully"
        
        # Restart nginx
        log "Restarting nginx..."
        docker compose -f /app/production.yml start nginx
        
        # Verify service is back up
        sleep 10
        if curl -s -f https://foudadev.site > /dev/null; then
            log "VERIFICATION: Site is accessible via HTTPS"
        else
            log "WARNING: Site verification failed"
            send_alert "SSL renewal completed but site verification failed"
        fi
        
    else
        log "ERROR: Certificate renewal failed"
        send_alert "SSL certificate renewal failed - manual intervention required"
        # Restart nginx anyway to restore service
        docker compose -f /app/production.yml start nginx
    fi
    
    log "=== SSL renewal process completed ==="
}

# Main execution
case "$1" in
    renew)
        renew_ssl
        ;;
    status)
        check_ssl_status
        ;;
    test)
        log "=== Dry run test ==="
        docker compose -f /app/production.yml stop nginx
        docker run --rm \
            -v "/etc/letsencrypt:/etc/letsencrypt" \
            -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
            -p 80:80 \
            certbot/certbot renew --dry-run
        docker compose -f /app/production.yml start nginx
        log "=== Dry run completed ==="
        ;;
    *)
        echo "Usage: $0 {renew|status|test}"
        echo "  renew  - Perform actual certificate renewal"
        echo "  status - Check current certificate status"
        echo "  test   - Dry run test without actual renewal"
        exit 1
        ;;
esac
```

#### Step 2: Make Executable
```bash
sudo chmod +x /root/ssl-renewal-advanced.sh
```

#### Step 3: Set up Crontab
```bash
sudo crontab -e
```

Add:
```bash
# SSL renewal with comprehensive logging
0 3 * * 1 /root/ssl-renewal-advanced.sh renew >> /var/log/ssl-renewal-advanced.log 2>&1

# Monthly certificate status report
0 2 1 * * /root/ssl-renewal-advanced.sh status >> /var/log/ssl-renewal-advanced.log 2>&1
```

#### Step 4: Test the System
```bash
# Test dry run
sudo /root/ssl-renewal-advanced.sh test

# Check status
sudo /root/ssl-renewal-advanced.sh status

# View logs
tail -f /var/log/ssl-renewal-advanced.log
```

**Pros:**
- Comprehensive error handling
- Notifications and alerts
- Status reporting
- Dry run capability
- Most robust solution

**Cons:**
- Complex setup
- Multiple dependencies
- Requires maintenance

---

## Verification & Testing

### Test Certificate Status
```bash
# Check certificate expiration
sudo docker run --rm \
  -v "/etc/letsencrypt:/etc/letsencrypt" \
  -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
  certbot/certbot certificates

# Check specific certificate dates
sudo openssl x509 -in /etc/letsencrypt/live/foudadev.site/cert.pem -noout -dates
```

### Test Manual Renewal
```bash
# For Method 1: Test the process manually
docker compose -f /app/production.yml stop nginx
docker run --rm -v "/etc/letsencrypt:/etc/letsencrypt" -v "/var/lib/letsencrypt:/var/lib/letsencrypt" -p 80:80 certbot/certbot renew --dry-run
docker compose -f /app/production.yml start nginx

# For Method 2: Test the script
sudo /root/renew-ssl.sh

# For Method 3: Test advanced features
sudo /root/ssl-renewal-advanced.sh test
sudo /root/ssl-renewal-advanced.sh status
```

### Verify Crontab
```bash
# List current crontab
sudo crontab -l

# Check cron execution logs
grep CRON /var/log/syslog | tail -20
```

---

## Troubleshooting

### Common Issues & Solutions

#### Issue: Port 80 is busy
```bash
# Check what's using port 80
sudo netstat -tulpn | grep :80

# Kill process if necessary
sudo fuser -k 80/tcp
```

#### Issue: Permission denied
```bash
# Fix certificate permissions
sudo chown -R root:root /etc/letsencrypt
sudo chmod -R 755 /etc/letsencrypt
```

#### Issue: Crontab not executing
```bash
# Check if cron service is running
sudo systemctl status cron

# Check crontab syntax
sudo crontab -l

# Check cron logs
sudo grep CRON /var/log/syslog
```

#### Issue: Certificates not found
```bash
# Verify certificate location
sudo ls -la /etc/letsencrypt/live/foudadev.site/

# Check nginx configuration
sudo docker exec -it nginx nginx -t
```

### Monitoring Commands
```bash
# Monitor renewal logs
tail -f /var/log/ssl-renewal.log

# Check certificate expiration regularly
sudo docker run --rm -v "/etc/letsencrypt:/etc/letsencrypt" -v "/var/lib/letsencrypt:/var/lib/letsencrypt" certbot/certbot certificates

# Verify website SSL
curl -I https://foudadev.site
```

---

## Recommendation Summary

- **For Beginners**: Use **Method 1** - Simple crontab approach
- **For Production**: Use **Method 2** - Script-based with logging  
- **For Critical Systems**: Use **Method 3** - Advanced with monitoring and alerts

**Quick Start Choice**: If you're unsure, start with **Method 1** and upgrade to **Method 2** later when you need better logging.

All methods will successfully auto-renew your SSL certificates and ensure your site remains secure and accessible.
