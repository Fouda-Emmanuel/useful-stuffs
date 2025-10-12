# 🚀 Complete Monitoring Setup Guide: Django + Prometheus + Grafana

*A comprehensive guide to setting up production-grade monitoring for Django applications using Prometheus and Grafana*

---

## 📖 Table of Contents
1. [Overview](#-overview)
2. [Architecture](#-architecture)
3. [Prerequisites](#-prerequisites)
4. [Django Setup](#-django-setup)
5. [Docker Configuration](#-docker-configuration)
6. [Monitoring Services](#-monitoring-services)
7. [Grafana Dashboard](#-grafana-dashboard)
8. [Testing & Verification](#-testing--verification)
9. [Troubleshooting](#-troubleshooting)
10. [Production Considerations](#-production-considerations)

---

## 🎯 Overview

This guide sets up a complete monitoring stack for Django applications that provides:

- **Application Metrics**: Request rates, latency, database queries, cache performance
- **System Metrics**: CPU, memory, disk, network usage
- **Container Metrics**: Docker container resource usage
- **Visualization**: Beautiful dashboards with Grafana
- **Alerting**: Configurable alerts (optional)

---

## 🏗️ Architecture

```
+-------------+    +-------------+    +-------------+
|   Django    |    | Prometheus  |    |   Grafana   |
|   App       |    |             |    |             |
| (api:8000)  |    | (9090)      |    | (3000)      |
| /metrics    +----> Scrapes     +----> Visualizes  |
+-------------+    +-------------+    +-------------+
                         ^
                         |
+-------------+          |
| Node        |          |
| Exporter    +----------+
| (9100)      |
| /metrics    |
+-------------+
```

---

## 📋 Prerequisites

- Docker & Docker Compose
- Django application
- Basic understanding of container networking

---

## 🐍 Django Setup

### 1. Install Dependencies

```bash
# Add to requirements.txt
django-prometheus==2.3.0
```

```bash
pip install django-prometheus
```

### 2. Configure Django Settings

**`settings.py`**:

```python
# Add to INSTALLED_APPS
INSTALLED_APPS = [
    # ... your existing apps ...
    "django_prometheus",
]

# Update MIDDLEWARE (order is important!)
MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',  # First
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'django_prometheus.middleware.PrometheusAfterMiddleware',   # Last
]

# Configure database for monitoring
DATABASES = {
    'default': {
        'ENGINE': 'django_prometheus.db.backends.postgresql',  # Changed!
        'NAME': getenv("POSTGRES_DB"),
        'USER': getenv("POSTGRES_USER"),
        'PASSWORD': getenv("POSTGRES_PASSWORD"),
        'HOST': getenv("POSTGRES_HOST"),
    }
}

# Configure cache for monitoring
CACHES = {
    "default": {
        "BACKEND": "django_prometheus.cache.backends.redis.RedisCache",  # Changed!
        "LOCATION": getenv("REDIS_URL", "redis://redis:6379/1"),
    }
}

# Optional: Custom metric namespace
PROMETHEUS_METRIC_NAMESPACE = getenv("PROMETHEUS_METRIC_NAMESPACE", "yourapp")
```

### 3. Add URL Routes

**`urls.py`**:

```python
from django.urls import include, path

urlpatterns = [
    # ... your existing URL patterns ...
    
    # Add Prometheus metrics endpoint
    path('', include('django_prometheus.urls')),
]
```

---

## 🐳 Docker Configuration

### 1. Docker Compose Setup

**`docker-compose.yml`**:

```yaml
version: '3.8'

services:
  # Your existing services
  api:
    build: .
    volumes:
      - .:/app
    expose:
      - "8000"  # Important: expose but don't publish
    environment:
      - PROMETHEUS_METRIC_NAMESPACE=yourapp
    networks:
      - monitoring_net

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=yourapp
      - POSTGRES_USER=yourapp
      - POSTGRES_PASSWORD=password
    networks:
      - monitoring_net

  redis:
    image: redis:7-alpine
    networks:
      - monitoring_net

  # Monitoring Services
  prometheus:
    image: prom/prometheus:latest
    container_name: yourapp-prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    networks:
      - monitoring_net
    depends_on:
      - api

  grafana:
    image: grafana/grafana:latest
    container_name: yourapp-grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    networks:
      - monitoring_net
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:latest
    container_name: yourapp-node-exporter
    command:
      - '--path.rootfs=/host'
    volumes:
      - /:/host:ro,rslave
    ports:
      - "9100:9100"
    networks:
      - monitoring_net

networks:
  monitoring_net:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
```

### 2. Prometheus Configuration

**`prometheus.yml`**:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'yourapp-api'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['api:8000']
    scrape_interval: 30s
    scrape_timeout: 10s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093
```

### 3. Grafana Provisioning (Optional)

Create **`grafana/provisioning/datasources/prometheus.yml`**:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

---

## 🚀 Deployment

### 1. Start the Stack

```bash
# Create external network (if not exists)
docker network create monitoring_net

# Start all services
docker-compose up -d

# Check running services
docker-compose ps
```

### 2. Verify Services

```bash
# Check all containers are running
docker ps

# Expected output:
# yourapp-prometheus
# yourapp-grafana  
# yourapp-node-exporter
# yourapp-api-1
# yourapp-postgres-1
# yourapp-redis-1
```

---

## 🔍 Testing & Verification

### 1. Test Metrics Endpoint

```bash
# Test from inside Docker network
docker run --rm -it --network yourapp_monitoring_net curlimages/curl:latest curl http://api:8000/metrics

# Expected: Prometheus metrics in text format
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
# python_gc_objects_collected_total{generation="0"} 1639.0
```

### 2. Check Prometheus Targets

Visit: `http://localhost:9090/targets`

**Expected Status**: 
- `yourapp-api` → **UP**
- `node-exporter` → **UP** 
- `prometheus` → **UP**

### 3. Access Grafana

Visit: `http://localhost:3000`

**Default Credentials**:
- Username: `admin`
- Password: `admin123`

### 4. Configure Grafana Data Source

1. Go to **Connections** → **Data sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set URL: `http://prometheus:9090`
5. Click **Save & Test** → Should show "Data source is working"

---

## 📊 Grafana Dashboard Setup

### 1. Create Application Metrics Dashboard

**Panel 1: HTTP Requests**
- **Query**: `rate(django_http_requests_total_by_method_total[1m])`
- **Legend**: `{{method}}`
- **Title**: HTTP Requests per Second

**Panel 2: Request Latency (P95)**
- **Query**: `histogram_quantile(0.95, rate(django_http_requests_latency_seconds_bucket[5m]))`
- **Title**: 95th Percentile Request Latency
- **Unit**: seconds

**Panel 3: Database Queries**
- **Query**: `rate(django_db_queries_total[1m])`
- **Title**: Database Queries per Second

**Panel 4: Cache Performance**
- **Query**: `rate(django_cache_get_total[1m])`
- **Title**: Cache Gets per Second

### 2. Create System Metrics Dashboard

**Panel 1: CPU Usage**
- **Query**: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)`
- **Title**: CPU Usage %
- **Unit**: percent

**Panel 2: Memory Usage**
- **Query**: `node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes`
- **Title**: Memory Used
- **Unit**: bytes

**Panel 3: Disk Usage**
- **Query**: `100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})`
- **Title**: Disk Usage %
- **Unit**: percent

**Panel 4: Network Traffic**
- **Query**: `rate(node_network_receive_bytes_total[1m])`
- **Title**: Network Received
- **Unit**: bytes/sec

### 3. Arrange Dashboard Layout

1. Click **Edit** on dashboard
2. Drag panels to create grid layout
3. Resize panels using bottom-right corner
4. Click **Save dashboard**

**Example Layout**:
```
+------------------+------------------+
| HTTP Requests    | Request Latency  |
+------------------+------------------+
| DB Queries       | Cache Performance|
+------------------+------------------+
| CPU Usage        | Memory Usage     |
+------------------+------------------+
```

---

## 🐛 Troubleshooting

### 1. Prometheus Targets Show "DOWN"

```bash
# Test connectivity to API
docker run --rm -it --network yourapp_monitoring_net curlimages/curl:latest curl -v http://api:8000/metrics

# Check API container logs
docker logs yourapp-api-1

# Verify network
docker network inspect yourapp_monitoring_net
```

### 2. Grafana "No Data" in Panels

- Verify Prometheus data source URL is correct
- Check time range in Grafana (set to "Last 1 hour")
- Ensure metric names match exactly
- Test query in Prometheus UI first: `http://localhost:9090`

### 3. Metrics Not Exposed by Django

```python
# Verify django-prometheus is in INSTALLED_APPS
# Check middleware order
# Confirm /metrics URL is included
# Test locally: python manage.py runserver + curl http://localhost:8000/metrics
```

### 4. Reset Grafana Password

```bash
docker exec -it yourapp-grafana grafana-cli admin reset-admin-password newpassword
docker restart yourapp-grafana
```

### 5. Clear All Data

```bash
# Stop and remove containers and volumes
docker-compose down -v

# Remove Docker network
docker network rm yourapp_monitoring_net
```

---

## 🔧 Advanced Configuration

### 1. Custom Application Metrics

**`monitoring/metrics.py`**:

```python
from django_prometheus.metrics import Metric
from prometheus_client import Counter, Histogram

# Custom business metrics
orders_created = Counter('yourapp_orders_created_total', 'Total orders created')
payment_processing_time = Histogram('yourapp_payment_processing_seconds', 'Payment processing time')

def record_order_creation():
    orders_created.inc()

def record_payment_time(start_time):
    payment_processing_time.observe(time.time() - start_time)
```

### 2. Alerting Rules

**`prometheus/alerts.yml`**:

```yaml
groups:
  - name: yourapp_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(django_http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(django_http_requests_latency_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.instance }}"
          description: "95th percentile latency is {{ $value }}s"
```

### 3. Database Monitoring

Add to **`prometheus.yml`**:

```yaml
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

---

## 🛡️ Production Considerations

### 1. Security

```yaml
# Add to docker-compose.yml
environment:
  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
  - GF_SECURITY_SECRET_KEY=${GRAFANA_SECRET_KEY}
```

### 2. Data Persistence

```yaml
volumes:
  prometheus_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /path/to/prometheus/data
  
  grafana_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /path/to/grafana/data
```

### 3. Resource Limits

```yaml
services:
  prometheus:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
  
  grafana:
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
```

### 4. Backup Strategy

```bash
# Backup Prometheus data
docker run --rm -v yourapp_prometheus_data:/data -v $(pwd):/backup alpine tar czf /backup/prometheus-backup-$(date +%Y%m%d).tar.gz /data

# Backup Grafana data  
docker run --rm -v yourapp_grafana_data:/data -v $(pwd):/backup alpine tar czf /backup/grafana-backup-$(date +%Y%m%d).tar.gz /data
```

---

## 📈 Monitoring Checklist

- [ ] Django metrics endpoint accessible
- [ ] Prometheus targets show UP
- [ ] Grafana connects to Prometheus
- [ ] Application metrics visible
- [ ] System metrics collecting
- [ ] Dashboards created and arranged
- [ ] Alerts configured (optional)
- [ ] Data persistence working
- [ ] Security measures in place

---

## 🎉 Conclusion

You now have a complete monitoring stack that provides:

✅ **Real-time visibility** into your Django application  
✅ **System resource monitoring**  
✅ **Beautiful, customizable dashboards**  
✅ **Historical data retention**  
✅ **Production-ready architecture**

This setup will help you identify performance bottlenecks, troubleshoot issues quickly, and ensure your application runs smoothly in production.

---

## 📚 Additional Resources

- [Django Prometheus Documentation](https://github.com/korfuri/django-prometheus)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Node Exporter GitHub](https://github.com/prometheus/node_exporter)

**Happy Monitoring!** 🚀
