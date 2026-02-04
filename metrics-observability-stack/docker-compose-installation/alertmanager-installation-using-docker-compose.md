# Alertmanager Installation on Linux Using Docker Compose

## Objective

The objective of this SOP is to define the standardized procedure for installing, configuring, and deploying **Alertmanager** using **Docker Compose** in a **Linux** environment. This ensures a consistent, secure, and repeatable alerting setup integrated with **Prometheus**.

---

## Prerequisites

* **Host OS** with **Docker Engine** installed and running.

  ```bash
  docker --version
  # or
  sudo systemctl status docker
  ```
* **Docker Compose** available (v2 plugin or docker-compose).

  ```bash
  docker compose version
  # or
  docker-compose --version
  ```
* **Shell access** (sudo/root) to create directories and manage permissions.
* **External Docker network** shared with Prometheus (example: `monitoring`).
* Sufficient **disk space** for Alertmanager state and logs.
* **Network access** to pull **Docker images**.

---

## Prepare The System

### Ensure System Packages Are Updated

* For **Debian/Ubuntu**

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

* For **RHEL/CentOS/Fedora**

  ```bash
  sudo yum update -y
  ```

#

### Create Dedicated User for the Service:

>**Note:** If the user has already been created during `Prometheus` or `Grafana` instllation then do not need to create the user again. If it has not then create the user by following the below procedure.

- Create observer user for this service
  ```bash
  sudo useradd observer
  ```
  
- Add observer user to docker group
  ```bash
  sudo usermod -aG docker observer
  ```
  
- Check user and group created or not
  ```bash
  id observer
  ```
  **Output:** It will output the user and group id with their name.

#

### Create Docker Network

* Alertmanager will use an **external Docker bridge network** shared with Prometheus.

  ```bash
  docker network create monitoring
  ```
  **Note:**
  This network allows **Prometheus, Alertmanager, and Grafana** to communicate internally without exposing services unnecessarily.

---

## Directory Structure

### Directory Structure for Alertmanager

* Recommended logical structure:

  ```
  /srv/alertmanager/
  ├── config/
  │   └── alertmanager.yml
  │
  ├── data/
  │   └── (Alertmanager state files)
  │
  ├── logs/
  │   └── (Alertmanager logs)
  │
  └── docker-compose.yml
  ```

* **Actual directory structure used in this SOP**:

  > Here Alertmanager is deployed inside a dedicated `container` directory under the user’s home directory.

  ```
  ~/container/alertmanager/
  ├── alertmanager_config/
  │   └── alertmanager.yml
  │
  ├── alertmanager_data/
  │   └── (Alertmanager state files)
  │
  ├── alertmanager_logs/
  │   └── (Alertmanager logs)
  │
  └── docker-compose.yml
  ```

---

## Create the Necessary Directories

* Navigate to the **Alertmanager** container directory:

  ```bash
  mkdir -p ~/container/alertmanager
  cd ~/container/alertmanager
  ```

* Create required directories:

  ```bash
  mkdir alertmanager_config alertmanager_data alertmanager_logs
  ```

* Verify directory creation:

  ```bash
  ls -ld alertmanager_config alertmanager_data alertmanager_logs
  ```

---

## Set Necessary Permissions

### Identify Alertmanager Container User

* Run a temporary Alertmanager container to check UID/GID:

  ```bash
  docker run --rm -it --entrypoint /bin/sh prom/alertmanager:latest -c "id"
  ```

* Expected output:

  ```
  uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)
  ```

### Apply Ownership on Host Directories

* Change ownership using the identified UID/GID:

  ```bash
  sudo chown -R 65534:65534 alertmanager_config alertmanager_data alertmanager_logs
  ```

---

## Create Alertmanager Configuration File 

Create the configuration file:

```
~/container/alertmanager/alertmanager_config/alertmanager.yml
```

Minimal configuration example:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'

receivers:
  - name: 'default'
    # Notification integrations (Email, Slack, Webhook, MS Teams) can be added here
    # Look for Notification integrations sop for this configurations.
```

**Note:**
This configuration is sufficient for initial deployment. Notification receivers and routing rules can be added later.

---

## Create `docker-compose` file

Create:

```
~/container/alertmanager/docker-compose.yml
```

Compose file:

```yaml
services:
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    user: "65534:65534"
    ports:
      - "192.168.0.106:9093:9093"
    volumes:
      - ./alertmanager_data:/alertmanager:z
      - ./alertmanager_config:/etc/alertmanager:ro,z
      - ./alertmanager_logs:/var/log/alertmanager:z
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
      - "--web.enable-lifecycle"
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9093/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.25"
          pids: 100
    networks:
      - monitoring

networks:
  monitoring:
    external: true
```

**Notes:**

* `:ro` ensures configuration files are not modified by the container.
* Consider pinning the image version (e.g., `v0.27.0`) for production stability.
* Resource limits help prevent alerting services from exhausting host resources.

---

## Firewall Adjustment

* Open port **9093** (example using firewalld):

  ```bash
  sudo firewall-cmd --add-port=9093/tcp --permanent
  sudo firewall-cmd --reload
  ```

---

## Start Alertmanager

* Using Docker Compose v2:

  ```bash
  docker compose up -d
  ```

* Using legacy docker-compose:

  ```bash
  docker-compose up -d
  ```

---

## Verify Alertmanager Deployment

### Check Container Status

```bash
docker ps --filter "name=alertmanager"
# or
docker compose ps
```

### View Logs

```bash
docker compose logs -f alertmanager
# or
docker logs -f alertmanager
```

### Health Check

```bash
docker inspect -f '{{.State.Health.Status}}' alertmanager
```

### Access UI

```
http://192.168.20.126:9093
```

---

## Notes

* **Default port:** `9093`
* **Configuration file:** `~/containers/alertmanager/alertmanager_config/alertmanager.yml`
* **Data path:** `~/containers/alertmanager/alertmanager_data`
* **Log path:** `~/containers/alertmanager/alertmanager_logs`
* Alertmanager must be referenced in **Prometheus `alertmanagers` configuration** in `prometheus.yml` file for alerts to be delivered.
* Look for **Prometheus ↔ Alertmanager integration SOP** for this configuration detials.

---
