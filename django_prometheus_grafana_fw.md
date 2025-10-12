# A FoudaWire Monitoring Setup with Prometheus & Grafana

## 📋 Overview
This document explains how to set up comprehensive monitoring for the F-W API using Prometheus for metrics collection and Grafana for visualization.

## 🛠️ Prerequisites
- Docker & Docker Compose
- Django application
- Basic understanding of container networking

## 🔧 Step 1: Install Django Prometheus

Add to your `requirements.txt`:
```txt
django-prometheus==2.3.0
```

Install:
```bash
pip install django-prometheus
```

## ⚙️ Step 2: Configure Django Settings

### Update `settings.py`:

```python
# Add to INSTALLED_APPS
INSTALLED_APPS = [
    # ... existing apps ...
    "django_prometheus",
]

# Update MIDDLEWARE (order matters!)
MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',  # First
    # ... other middleware ...
    'django_prometheus.middleware.PrometheusAfterMiddleware',   # Last
]

# Update DATABASES for PostgreSQL monitoring
DATABASES = {
    'default': {
        'ENGINE': 'django_prometheus.db.backends.postgresql',
        'NAME': getenv("POSTGRES_DB"),
        'USER': getenv("POSTGRES_USER"),
        'PASSWORD': getenv("POSTGRES_PASSWORD"),
        'HOST': getenv("POSTGRES_HOST"),
    }
}

# Update CACHES for Redis monitoring
CACHES = {
    "default": {
        "BACKEND": "django_prometheus.cache.backends.redis.RedisCache",
        "LOCATION": getenv("REDIS_URL", "redis://redis:6379/1"),
    }
}

# Prometheus configuration
PROMETHEUS_METRIC_NAMESPACE = getenv("PROMETHEUS_METRIC_NAMESPACE", "foudawire")
```

### Update `urls.py`:
```python
from django.urls import include, path

urlpatterns = [
    # ... your existing URLs ...
    path('', include('django_prometheus.urls')),  # Adds /metrics endpoint
]
```

## 🐳 Step 3: Docker Compose Configuration

### Update `local.yml`:

```yaml
services:
  # Your existing services (api, postgres, redis, etc.)
  
  prometheus:
    image: prom/prometheus:latest
    container_name: foudawire-prometheus
    volumes:
      - ./docker/local/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - api
    networks:
      - banker_local_nw

  grafana:
    image: grafana/grafana:latest
    container_name: foudawire-grafana
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    env_file:
      - ./.envs/.env.local
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - banker_local_nw

  node-exporter:
    image: prom/node-exporter:latest
    container_name: foudawire-node-exporter
    command:
      - '--path.rootfs=/host'
    volumes:
      - /:/host:ro,rslave
    ports:
      - "9100:9100"
    networks:
      - banker_local_nw

networks:
  banker_local_nw:
    external: true

volumes:
  grafana_data:
```

### Create Prometheus Configuration

Create `docker/local/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 60s

scrape_configs:
  - job_name: "foudawire"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["api:8000"]

  - job_name: 'node-exporter'
    metrics_path: "/metrics"
    static_configs:
      - targets: ['node-exporter:9100']
```

## 🚀 Step 4: Start the Monitoring Stack

```bash
# Start all services
docker compose -f local.yml up -d

# Check running containers
docker ps
```

Expected output should show:
- `foudawire-prometheus`
- `foudawire-grafana` 
- `foudawire-node-exporter`
- Your API and other services

## 🔍 Step 5: Verify Metrics Endpoint

### Test from inside Docker network:
```bash
# Use temporary curl container to test metrics
docker run --rm -it --network banker_local_nw curlimages/curl:latest curl http://api:8000/metrics
```

Expected output: Prometheus metrics in text format like:
```
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 1639.0
...
```

### Check Prometheus Targets:
Visit `http://localhost:9090/targets` - both targets should show **UP** status.

## 🎨 Step 6: Configure Grafana

### Access Grafana:
- URL: `http://localhost:3000`
- Default credentials: `admin/admin123`

### Reset admin password (if needed):
```bash
docker exec -it foudawire-grafana grafana-cli admin reset-admin-password admin123
docker restart foudawire-grafana
```

### Add Prometheus Data Source:
1. Go to **Connections** → **Data sources** → **Add data source**
2. Select **Prometheus**
3. Set URL: `http://foudawire-prometheus:9090`
4. Click **Save & Test** - should show "Data source is working"

## 📊 Step 7: Create Dashboard

### Create New Dashboard:
1. **Dashboards** → **New** → **New Dashboard**
2. **Add visualization** for each metric

### Essential Metrics Queries:

#### Application Metrics:
```promql
# HTTP Requests per minute
rate(foudawire_django_http_requests_latency_seconds_by_view_method_count[1m])

# DB Queries per minute  
rate(foudawire_django_db_execute_total[1m])

# Average DB Query Duration
foudawire_django_db_query_duration_seconds_sum / foudawire_django_db_query_duration_seconds_count
```

#### System Metrics:
```promql
# CPU Usage %
100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])))

# Memory Usage
node_memory_Active_bytes

# Disk Usage
100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})

# Network Traffic
rate(node_network_receive_bytes_total[1m])
rate(node_network_transmit_bytes_total[1m])
```

### Arrange Panels Side by Side:
1. Click **Edit** on dashboard
2. Drag panels to position them horizontally
3. Resize panels using bottom-right corner
4. **Save dashboard**

## 🐛 Troubleshooting Common Issues

### 1. Prometheus Target Shows "DOWN"
```bash
# Check if API metrics endpoint is accessible
docker exec -it api-api-1 curl http://localhost:8000/metrics

# If curl not available, use Python
docker exec -it api-api-1 python -c "import requests; print(requests.get('http://localhost:8000/metrics').text)"
```

### 2. Grafana Login Issues
```bash
# Reset Grafana admin password
docker exec -it foudawire-grafana grafana-cli admin reset-admin-password admin123
docker restart foudawire-grafana

# Or reset completely by removing volume
docker compose -f local.yml down
docker volume rm api_grafana_data
docker compose -f local.yml up -d
```

### 3. Metrics Not Showing in Grafana
- Verify Prometheus data source URL is `http://foudawire-prometheus:9090`
- Check time range in Grafana (set to "Last 1 hour")
- Ensure metrics names match exactly what's exposed

### 4. Container Network Issues
```bash
# Verify containers are on same network
docker network inspect banker_local_nw

# Test connectivity between containers
docker run --rm -it --network banker_local_nw curlimages/curl:latest curl http://api:8000/metrics
```

## 📈 Useful Grafana Dashboard IDs

Import pre-built dashboards:
- **Node Exporter Full**: ID `1860`
- **Prometheus 2.0 Overview**: ID `3662` 
- **Django Monitoring**: Search Grafana.com for Django-specific dashboards

## 🎯 Key Metrics to Monitor

### Application Level:
- HTTP request rate and latency
- Database query performance  
- Cache hit ratios
- Celery task queue lengths

### System Level:
- CPU and memory usage
- Disk I/O and space
- Network traffic
- Container resource usage

## ✅ Verification Checklist

- [ ] Django /metrics endpoint returns data
- [ ] Prometheus targets show UP status
- [ ] Grafana connects to Prometheus data source
- [ ] Dashboard panels show live data
- [ ] All containers running without errors
- [ ] Time series graphs updating in real-time

This setup provides comprehensive monitoring for your bank system API, giving you visibility into both application performance and infrastructure health.


### ----------------------------------------------------------------------------------------------------------------------------------------------
### ----------------------------------------------------------------------------------------------------------------------------------------------

# B 🧠 Deep Dive: Docker Networking & Testing Metrics Endpoint

You're absolutely right to question this! Let me explain the **why** behind that command and the networking concepts.

## 🔍 Why We Need to Test `http://api:8000/metrics`

### The Problem:
When you try to access `http://api:8000/metrics` from your **host machine** (your laptop), it fails with:
```
DNS_PROBE_FINISHED_NXDOMAIN
```

### The Reason:
- `api` is a **Docker internal hostname** 
- It only exists **inside the Docker network** (`banker_local_nw`)
- Your host machine's DNS doesn't know about `api` - it's not a real domain!

## 🌐 Docker Networking Explained

### Your Current Setup:
```yaml
# In your docker-compose
services:
  api:
    container_name: api-api-1
    networks:
      - banker_local_nw

  prometheus:
    container_name: foudawire-prometheus  
    networks:
      - banker_local_nw

networks:
  banker_local_nw:
    external: true
```

### What Happens Inside Docker:
```
+----------------------+
| Docker Network       |
| banker_local_nw      |
|                      |
|  api-api-1           |  ← Can be reached as "api" 
|  (port 8000)         |     (Docker DNS magic!)
|                      |
|  foudawire-prometheus|  ← Can reach "api:8000"
+----------------------+
```

### What Happens Outside Docker:
```
Your Host Machine (laptop)
↓ tries to resolve "api"
↓ Your ISP DNS: "I don't know api!"
↓ DNS_PROBE_FINISHED_NXDOMAIN ❌
```

## 🛠️ The Testing Command Breakdown

```bash
docker run --rm -it --network banker_local_nw curlimages/curl:latest curl http://api:8000/metrics
```

Let's break it down piece by piece:

### 1. `docker run`
- Starts a new temporary container

### 2. `--rm`
- **Remove the container after it exits** - keeps your system clean

### 3. `-it`
- **Interactive terminal** - so you can see the output

### 4. `--network banker_local_nw`
- **Crucial part!** Connects this container to your app's network
- Now it can see `api` as a valid hostname

### 5. `curlimages/curl:latest`
- Uses a minimal Docker image that has `curl` pre-installed
- Your API container doesn't have `curl` (it's a minimal Python image)
- Prometheus container doesn't have `curl` either

### 6. `curl http://api:8000/metrics`
- Tests the metrics endpoint **from inside the Docker network**
- This is exactly how Prometheus scrapes your metrics!

## 🎯 Why This Testing Approach is Better

### Option A: Bad Approach (Modify Your API Container)
```bash
# Don't do this - modifies your production container!
docker exec -it api-api-1 apt-get update && apt-get install curl
docker exec -it api-api-1 curl http://localhost:8000/metrics
```
**Problems:**
- Modifies your app container
- Adds unnecessary packages
- Breaks container immutability principle

### Option B: Better Approach (Temporary Container)
```bash
# Do this - clean and safe!
docker run --rm -it --network banker_local_nw curlimages/curl:latest curl http://api:8000/metrics
```
**Benefits:**
- ✅ Doesn't modify your app containers
- ✅ Tests the exact same network conditions as Prometheus
- ✅ Clean - container disappears after use
- ✅ Reproducible

## 🔄 Real-World Flow Example

### How Prometheus Actually Scrapes:
```
Prometheus Container
    ↓ (Inside Docker network)
    requests http://api:8000/metrics
    ↓
API Container
    ↓
Returns: 
# HELP django_http_requests_total ...
# TYPE django_http_requests_total counter
django_http_requests_total{method="GET"} 42
```

### How We Simulate This for Testing:
```
Temporary Curl Container
    ↓ (Also inside Docker network)  
    requests http://api:8000/metrics
    ↓
API Container
    ↓
Returns same metrics to us for verification
```

## 🧪 Alternative Testing Methods

### Method 1: Expose API Port (Not Recommended for Production)
```yaml
# In docker-compose - makes metrics publicly accessible
services:
  api:
    ports:
      - "8000:8000"  # Exposes to host
```
Then test with:
```bash
curl http://localhost:8000/metrics  # Now works from host
```

### Method 2: Use Nginx Proxy (More Secure)
Add to your Nginx config:
```nginx
location /metrics {
    proxy_pass http://api:8000/metrics;
}
```
Then test with:
```bash
curl http://localhost:8080/metrics  # Through Nginx
```

### Method 3: Our Approach (Recommended)
```bash
docker run --rm -it --network banker_local_nw curlimages/curl:latest curl http://api:8000/metrics
```

## 📋 When Do You Need This Testing?

1. **Initial Setup** - Verify metrics endpoint works
2. **After Changes** - Confirm new metrics are exposed  
3. **Troubleshooting** - When Prometheus shows "DOWN"
4. **Debugging** - Check if specific metrics exist

## 🎓 Key Takeaways

1. **Docker DNS Magic**: Container names become hostnames within Docker networks
2. **Network Isolation**: Services can talk to each other but are isolated from host
3. **Testing Strategy**: Use temporary containers to test internal endpoints
4. **Security**: Keeping metrics internal is more secure than exposing them

## ❓ Common "Why" Questions Answered

### Q: "Why can't I just use `localhost`?"
**A**: `localhost` inside a container refers to that container itself, not your host machine.

### Q: "Why not install curl in my API container?"
**A**: It breaks the principle of minimal containers and adds attack surface.

### Q: "Is there a way to make `api` resolvable from my host?"
**A**: Yes, but it's complex and not recommended. You'd need to modify your host's DNS settings.

This approach ensures you're testing in the exact same environment that Prometheus uses, without modifying your production containers! 🚀
