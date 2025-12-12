# Prometheus Installation on Linux Using Docker Compose

## Objective
The objective of this SOP is to define the standardized procedure for installing, configuring, and deploying **Prometheus** using **Docker Compose** in a **Linux** environment. This ensures a consistent, reliable, and repeatable setup to support monitoring operations.

## Prerequisites
- **Host OS** with **Docker Engine** installed and running.
  **Verify:**\
  `docker --version` and `sudo systemctl status docker`.
  
- **Docker Compose** available (v2 plugin or docker-compose).
  **Verify:**\
  `docker compose version` or `docker-compose --version`.
  
- Sufficient **disk space** for Prometheus data (default retains TSDB on host).
- **Shell access** (sudo/root) to create directories, update firewall, change SELinux context if required.

## Prepare The System

### Ensure System Packages are Updated:

- Update and upgrade packages for **Debian/Ubuntu** user
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- Update and upgrade packages for **RHEL/CentOS/Fedora** user
  ```bash
  sudo yum update -y
  ```

### Create Dedicated User for the Service:

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

### Final Directory Structure

- The directory structure should be like this.

  ```
  /srv/prometheus/
  ├── config/
  │   └── prometheus.yml
  │
  ├── data/
  │   └── (Prometheus TSDB files)
  │
  ├── logs/
  │   └── (empty unless configured)
  │
  └── docker-compose.yml
  ```

- But the directory structure I am using is like this:

  >Actually here I am using a dedicated user called **observer** and inside the observer **home** directory there is a dedicated **container** directory and I am going to create all the **necessary directory** there needed for **prometheus**. 

  ```
  /home/observer/container/prometheus/
  ├── prom_config/
  │   └── prometheus.yml
  │
  ├── prom_data/
  │   └── (Prometheus TSDB files)
  │
  ├── prom_logs/
  │   └── (empty unless configured)
  │
  └── docker-compose.yml
  ```

### Create the Necessary Directories:

- First **go** to prometheus directory
  ```bash
  cd /home/observer/container/prometheus
  ```
- Now **Create directory** for **configuration** file under prometheus directory
  ```bash
  mkdir prom_config
  ```
- Then Create Prometheus **TSDB time-series** data directory
  ```bash
  mkdir prom_data
  ```
- And last Create the **Prometheus Logs** directory
  ```bash
  mkdir prom_logs
  ```
**We can Create all of it in one Command**
- All in one Command:
  ```bash
  mkdir prom_config prom_data prom_logs
  ```
  
- To verify file ownership:
  ```bash
  ls -ld prom_config prom_data prom_logs
  ```

### Set Necessary Permission
- Verify user and ID information by running temporary prometheus container:

  ```bash
  docker run --rm -it --entrypoint /bin/sh prom/prometheus:latest
  ```
- Then run `whoami` and `id` command inside container
  ```
  /prometheus $ whoami
  Output: nobody
  
  /prometheus $ id nobody
  Output: uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)
  ```

- Now change the directory permission with this **user id = 65534**
  ```bash
  chown -R 65534:65534 prom_config prom_data prom_logs
  ```

### Create Network for Docker Container

- Here we will use a **custom bridge network** for prometheus docker container
  ```bash
  docker network create monitoring
  ```

## Create prometheus.yml (Prometheus config)

Create `container/prometheus/prom_config/prometheus.yml` with this minimal configuration (Prometheus will at least scrape itself):

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15   seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager_ip:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:

# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          name: "prometheus"
```

>**Note:** Save the file. (Add extra scrape jobs later as needed.)


## Create docker-compose.yml

Create `/container/prometheus/docker-compose.yml` with this content:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    user: 65534:65534
    ports:
      - "192.168.0.106:9090:9090"
    restart: unless-stopped
    volumes:
      - ./prom_config:/etc/prometheus:ro,z
      - ./prom_data:/prometheus:z
      - ./prom_logs:/var/log/prometheus:z
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
    external: true
```

**Notes:**
- `:ro` on the config file prevents Prometheus from modifying the host copy.
- Change `prom/prometheus:latest` to a pinned tag (e.g., v2.45.0) for **production stability**.

## Firewall Adjustment (if host firewall limits ports)
- Open port **9090** (example for firewalld):
  ```bash
  sudo firewall-cmd --add-port=9090/tcp --permanent
  sudo firewall-cmd --reload
  ```

## Start Prometheus
From the project directory:

- Using **Docker Compose v2 plugin**:
  ```bash
  sudo docker compose up -d
  ```

- Or using legacy **docker-compose**:
  ```bash
  sudo docker-compose up -d
  ```

**Note:** You can add `-f /path/to/docker-compose.yml` if running from another folder.

## Verify service is running
- Check containers:
  ```bash
  sudo docker compose ps
  # or
  sudo docker ps --filter "name=prometheus"
  ```

- View logs:
  ```bash
  sudo docker compose logs -f prometheus
  # or
  sudo docker logs -f prometheus
  ```

- Health check (from host or browser):
  - **Browser:**
      ```
      http://192.168.0.106:9090/
      # Prometheus UI home.
      ```
  - **Curl:**
      ```bash
      curl -sSf http://192.168.0.106:9090/-/ready
      # should return 200 if ready
      ```

## Notes
- **Default port:** `9090`
- **Configuration file:** `/home/observer/container/prometheus/prom_config/prometheus.yml`
- **Data path:** `/home/observer/container/prometheus/prom_data`
- **Log path:** `/home/observer/container/prometheus/prom_logs`
