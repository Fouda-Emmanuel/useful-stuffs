# **Foudawire Production Deployment Documentation**

## **Project Overview**

**Foudawire** is a comprehensive banking system API built with modern microservices architecture, containerized using Docker and orchestrated with Docker Compose.

### **Tech Stack**
- **Backend**: Django & Django REST Framework
- **Database**: PostgreSQL
- **Message Broker**: RabbitMQ
- **Caching**: Redis
- **Task Queue**: Celery with Celery Beat
- **Monitoring**: Flower, Prometheus, Grafana
- **Reverse Proxy**: Nginx with SSL
- **Containerization**: Docker & Docker Compose
- **Container Management**: Portainer

---

## **Infrastructure Architecture**

```
Internet → Nginx (SSL Termination) → API Services
                                  → Flower (Monitoring)
                                  → Portainer (Management)
                                  → Static Files

Backend Services:
- PostgreSQL (Database)
- Redis (Cache & Celery Result Backend)
- RabbitMQ (Message Broker)
- Celery Workers (Async Task Processing)
- Celery Beat (Scheduled Tasks)
```

---

## **Deployment Timeline & Steps**

### **1. Server Setup**
- **EC2 Instance**: Ubuntu 22.04 LTS (t2.medium)
- **Security Groups**: Open ports 22, 80, 443, 5555, 8000, 9443

### **2. Domain Configuration**
```
foudadev.site (Main Domain)
├── api.foudadev.site (API Endpoints)
├── flower.foudadev.site (Celery Monitoring)
└── portainer.foudadev.site (Container Management)
```

### **3. Docker Environment Setup**

#### **Network Creation**
```bash
docker network create foudawire_prod_nw
docker network ls
```

#### **Volume Management**
```bash
docker volume create static_volume
docker volume create app_logs
docker volume create rabbitmq_data
docker volume create flower_db
docker volume create grafana_data
docker volume create foudawire_prod_db
```

### **4. Service Deployment**

#### **Core Services Started**
```bash
docker-compose -f production.yml up -d
```

**Running Services:**
- ✅ app-postgres-1 (PostgreSQL Database)
- ✅ app-redis-1 (Redis Cache)
- ✅ app-rabbitmq-1 (RabbitMQ Broker)
- ✅ app-api-1 (Django API)
- ✅ app-celeryworker-1 (Celery Worker)
- ✅ app-celerybeat-1 (Celery Beat Scheduler)
- ✅ app-flower-1 (Celery Monitoring)
- ✅ app-nginx-1 (Nginx Reverse Proxy)
- ✅ portainer (Container Management)

---

## **Critical Issues & Resolutions**

### **Issue 1: Nginx Upstream Resolution Failure**

**Problem:**
```bash
2025/11/01 09:31:51 [emerg] 1#1: host not found in upstream "portainer:9443"
2025/11/01 09:32:18 [emerg] 1#1: host not found in upstream "api:8000"
```

**Root Cause:** Containers on different Docker networks

**Solution:**
```bash
# Connect all services to the same network
docker network connect foudawire_prod_nw portainer
docker network connect foudawire_prod_nw app-api-1
docker network connect foudawire_prod_nw app-flower-1

# Verify network connections
docker inspect portainer | grep -i network
```

### **Issue 2: Flower Startup Failures**

**Problem:** Celery workers not ready, authentication issues

**Solution:**
- Implemented wait loop in `start-flower.sh`
- Fixed environment variable parsing
- Used alphanumeric passwords to avoid special character issues

**Fixed start-flower.sh:**
```bash
#!/bin/bash
set -o errexit
set -o nounset

worker_ready(){
  celery -A config.celery_app inspect ping
}

until worker_ready; do
  echo >&2 'Celery workers not available'
  sleep 1
done

echo >&2 'Celery workers available'

exec celery \
  -A config.celery_app \
  -b "${CELERY_BROKER_URL}" \
  flower \
  --basic_auth="${CELERY_FLOWER_USER}:${CELERY_FLOWER_PASSWORD}"
```

### **Issue 3: Monitoring Stack Connectivity**

**Problem:** Prometheus, Grafana, Node Exporter containers exiting

**Resolution:** Network configuration and service dependencies

---

## **Final Service Status**

### **✅ Operational Services**
| Service | Container Name | Status | Ports |
|---------|----------------|---------|-------|
| Nginx | app-nginx-1 | ✅ Running | 80, 443 |
| API | app-api-1 | ✅ Running | 8000 |
| Flower | app-flower-1 | ✅ Running | 5555 |
| Celery Worker | app-celeryworker-1 | ✅ Running | - |
| Celery Beat | app-celerybeat-1 | ✅ Running | - |
| PostgreSQL | app-postgres-1 | ✅ Running | 5432 |
| Redis | app-redis-1 | ✅ Running | 6379 |
| RabbitMQ | app-rabbitmq-1 | ✅ Running | 5672, 15672 |
| Portainer | portainer | ✅ Running | 8000, 9443 |

### **🔄 Monitoring Services** (Requiring additional configuration)
| Service | Status | Issue |
|---------|---------|-------|
| Prometheus | ❌ Exited | Network configuration |
| Grafana | ❌ Exited | Network configuration |
| Node Exporter | ❌ Exited | Network configuration |

---

## **Access URLs**

- **API**: https://api.foudadev.site
- **Flower Dashboard**: https://flower.foudadev.site:5555
- **Portainer**: https://portainer.foudadev.site:9443
- **RabbitMQ Management**: http://your-ip:15672

---

## **Key Configuration Files**

### **Nginx Config Snippet**
```nginx
upstream api {
    server api:8000;
}

upstream flower {
    server flower:5555;
}

upstream portainer {
    server portainer:9443;
}

server {
    listen 443 ssl;
    server_name api.foudadev.site;
    
    ssl_certificate /etc/letsencrypt/live/foudadev.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/foudadev.site/privkey.pem;
    
    location / {
        proxy_pass http://api;
        include proxy_params;
    }
}
```

### **Docker Compose Network Section**
```yaml
networks:
  foudawire_prod_nw:
    external: true
    name: foudawire_prod_nw
```

---

## **Lessons Learned**

### **Docker Networking**
- All interconnected containers must be on the same Docker network
- Use `docker network connect` to fix network isolation issues
- Verify network membership with `docker inspect`

### **Service Dependencies**
- Implement health checks and wait loops for dependent services
- Flower requires Celery workers to be ready before starting
- Nginx requires all upstream services to be network-accessible

### **Environment Management**
- Use `.env` files for sensitive configuration
- Avoid special characters in environment variables
- Test service connectivity within the Docker network

### **Troubleshooting Workflow**
1. Check container status: `docker ps -a`
2. Examine logs: `docker logs <container>`
3. Verify networking: `docker network inspect`
4. Test connectivity: `docker exec -it container ping service`

---

## **Future Improvements**

1. **Service Mesh**: Implement Traefik for dynamic reverse proxying
2. **Health Checks**: Add Docker healthchecks to all services
3. **Logging**: Centralized logging with ELK stack
4. **Monitoring**: Fix and integrate Prometheus/Grafana stack
5. **CI/CD**: Automated deployment pipeline
6. **Backup**: Automated database and volume backups

---
