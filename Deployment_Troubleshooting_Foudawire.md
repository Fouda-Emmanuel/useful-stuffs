# **Foudawire Deployment Fix**

**Purpose:** Record all issues, errors, commands, and fixes with `nginx`, `portainer` and `flower` during production deployment for reference in future deployments.

---

## **1. Initial Setup**

* **Domain:** `foudadev.site`
* **DNS Provider:** Spaceship (nameservers: `launch1.spaceship.net`, `launch2.spaceship.net`)
* **EC2 Server:** Ubuntu, Docker, Docker Compose
* **Services:** Django API, Celery workers, Flower, NGINX, Postgres, Redis, RabbitMQ, Prometheus, Grafana, Portainer

**DNS Records:**

| Host      | Type | Value              |
| --------- | ---- | ------------------ |
| @         | A    | 3.95.55.53         |
| api       | A    | 3.95.55.53         |
| flower    | A    | 3.95.55.53         |
| portainer | A    | 3.95.55.53         |
| mg        | TXT  | brevo verification |

---

## **2. Deployment Errors and Fixes**

### **2.1 NGINX container not starting**

**Symptom:**
`app-nginx-1` container was constantly restarting.

**Error log:**

```
nginx: [emerg] host not found in upstream "flower:5555"
nginx: [emerg] host not found in upstream "portainer:9443"
nginx: [emerg] host not found in upstream "api:8000"
```

**Root Cause:**

* NGINX container could not resolve service names (`flower`, `api`, `portainer`) because they were not on the same Docker network.

**Commands to inspect:**

```bash
docker network ls
docker inspect portainer | grep -i network
docker ps
```

**Solution:**

* Connect `portainer` container to the `foudawire_prod_nw` network:

```bash
docker network connect foudawire_prod_nw portainer
```

* Verify with:

```bash
docker inspect portainer | grep -i foudawire_prod_nw -A 5
```

* Restart NGINX:

```bash
docker compose -f production.yml restart nginx
```

**Outcome:**

* NGINX started successfully.
* Domain `foudadev.site` and API endpoints became accessible.

---

### **2.2 Flower dashboard showing 502 Bad Gateway**

**Symptom:**
`flower.foudadev.site` returned a `502 Bad Gateway`.

**Error log inside Flower container:**

```
/start-flower.sh: line 7: -A: command not found
Celery workers not available
```

**Root Cause:**

* Bash script `/start-flower.sh` was calling `-A config.celery_app inspect ping` without the `celery` command.
* Extra quotation mark at `--basic_auth` caused syntax error.

**Original `/start-flower.sh`:**

```bash
worker_ready(){
  -A config.celery_app inspect ping
}

exec celery \
  -A config.celery_app \
  -b "${CELERY_BROKER_URL}" \
  flower \
  --basic_auth="${CELERY_FLOWER_USER}:${CELERY_FLOWER_PASSWORD}\""
```

**Corrected `/start-flower.sh`:**

```bash
#!/bin/bash

set -o errexit
set -o nounset

worker_ready() {
  celery -A config.celery_app inspect ping
}

until worker_ready; do
  echo >&2 'Celery workers not available'
  sleep 3
done

echo >&2 'Celery workers available'

exec celery \
  -A config.celery_app \
  -b "${CELERY_BROKER_URL}" \
  flower \
  --basic_auth="${CELERY_FLOWER_USER}:${CELERY_FLOWER_PASSWORD}"
```

**Commands to rebuild and restart Flower:**

```bash
docker compose -f production.yml up -d --build flower
docker logs app-flower-1 -f
```

**Outcome:**

* Flower dashboard became accessible at `https://flower.foudadev.site`.

---

### **2.3 HTTPS issues**

* Portainer was accessible via `https://3.95.55.53:9443` but browser marked it as "not secure."
* Root cause: Let's Encrypt certificates were applied to NGINX, not Portainer directly.
* Solution: Either secure Portainer via a reverse proxy on NGINX or access via HTTP internally.

---

## **3. Key Commands Used During Troubleshooting**

```bash
# Inspect containers
docker ps -a
docker logs app-nginx-1 --tail 50

# Check network connectivity
docker network ls
docker inspect portainer | grep -i network
docker network connect foudawire_prod_nw portainer

# Restart containers
docker compose -f production.yml restart nginx
docker compose -f production.yml up -d --build flower

# Verify certificates
sudo ls /etc/letsencrypt/live/foudadev.site

# Ping other containers
docker exec -it app-nginx-1 ping flower
```

---

## **4. Lessons Learned / Recommendations**

1. **Docker Networking**

   * Always ensure all upstream services in NGINX are on the same Docker network.
   * Use `depends_on` and network configs consistently in `docker-compose` or `production.yml`.

2. **Script Validation**

   * Always test entrypoint scripts in the container (`docker exec -it <container> bash`) before deployment.
   * Ensure all commands exist and paths are correct.

3. **Flower / Celery**

   * Use `celery -A <app> inspect ping` to confirm workers are available before starting Flower.
   * Avoid extra quotes in `--basic_auth` parameters.

4. **NGINX Reverse Proxy**

   * Map all services behind NGINX using upstreams and correct `server_name`.
   * Redirect HTTP to HTTPS, ensure SSL certificates are mounted.

5. **Portainer / Other Dashboards**

   * For production, secure access with NGINX or firewall rules.

6. **Documentation**

   * Keep this log for all future deployments. Copy `.env` values, networks, volumes, and commands.

---

### **5. References**

* Docker Networking: [https://docs.docker.com/network/](https://docs.docker.com/network/)
* Celery Flower Docs: [https://flower.readthedocs.io](https://flower.readthedocs.io)
* NGINX Reverse Proxy: [https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

---

