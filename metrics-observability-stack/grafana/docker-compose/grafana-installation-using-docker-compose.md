# Grafana Installation on Linux Using Docker Compose

## **Objective**
The objective of this SOP is to define the standardized procedure for installing, configuring, and deploying **Grafana** using **Docker Compose** in a **Linux environment**. This ensures a consistent, reliable, and repeatable setup to support monitoring and visualization operations.

## **Prerequisites**
- **Host OS** with **Docker Engine** installed and running.
  ```bash
  docker --version
  # or
  sudo systemctl status docker
  ```
- **Docker Compose** available (v2 plugin or docker-compose).
  ```bash
  docker compose version
  # or
  docker-compose --version
  ```
- **Sufficient disk space** for persistent Grafana data.
- **Shell access (sudo/root)** to create directories, set permissions, and adjust firewall if required.

---

## **Prepare the System**

### **Update Packages**
- **For Debian/Ubuntu:**
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- **For RHEL/CentOS/Fedora:**
  ```bash
  sudo yum update -y
  ```

### **Create a Dedicated User:**
- Create **observer** user for this service
  ```bash
  sudo useradd observer
  ```
- Add **observer** user to **docker group**
  ```bash
  sudo usermod -aG docker observer
  ```
- **Check** user and group created or not
  ```bash
  id observer
  ```
  **Output:** It will output the **user and group id** with their name.

### **Create Docker Network**
- Here we will use a custom bridge network for grafana docker container
  ```bash
  docker network create monitoring
  ```
  **Note:** If already created for Prometheus, skip it.

---

## Directory Structure

### Directory Structure for Grafana

- Preferred structure under **observer’s home directory**:
  ```
  /home/observer/container/grafana/
  ├── grafana_data/
  │   └── (SQLite DB, dashboards, plugin data)
  │
  ├── provisioning/
  │   └── (provisioning for automation)
  │
  ├── grafana_logs/
  │   └── (Grafana logs only if configured)
  │
  └── docker-compose.yml
  ```

### **Create Necessary Directories**
- Switch to Grafana directory:
  ```bash
  cd /home/observer/container/grafana
  ```
- Create required directories:
  ```bash
  mkdir grafana_data grafana_logs provisioning
  ```
- Verify:
  ```bash
  ls -ld grafana_data grafana_logs provisioning
  ```

### **Set Necessary Permissions**
Grafana container runs as:
  `uid=472(grafana) gid=472(grafana)`
  
- Verify user and ID by running **temporary** grafana container:
  ```bash
  docker run --rm -it --entrypoint /bin/sh grafana/grafana:latest
  ```

- Then run `whoami` and `id` command inside container
  ```
  /grafana $ whoami
  # grafana
  
  /grafana $ id grafana
  # uid=472(grafana) gid=472(grafana)
  ```
- Now apply appropriate permissions:
  ```bash
  sudo chown -R 472:472 grafana_data grafana_logs provisioning
  ```
**Note:** This ensures Grafana can write dashboards, plugins, and its internal SQLite database.


---

## **Create docker-compose.yml File**
- Create the file:
  ```bash
  nano /home/observer/container/grafana/docker-compose.yml
  ```
- Insert this **Grafana Compose file**:

  ```yaml
  services:
    grafana:
      image: grafana/grafana:latest
      container_name: grafana
      user: "472:472"
      ports:
        - "192.168.0.106:3000:3000"
      restart: unless-stopped
      volumes:
        - ./grafana_data:/var/lib/grafana:z
        - ./provisioning:/etc/grafana/provisioning:ro,z
        - ./grafana_logs:/var/log/grafana:z
      environment:
        - GF_SECURITY_ADMIN_USER=admin
        - GF_SECURITY_ADMIN_PASSWORD=admin
      healthcheck:
        test: ["CMD", "wget", "--spider", "-q", "http://localhost:9093/-/healthy"]
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s
      networks:
        - monitoring
  
  networks:
    monitoring:
      driver: bridge
      external: true
  ```
**Notes:**
- **Replace admin/admin in production.**
- Pin version for stability: `image: grafana/grafana:11.2.0`

---

## **Firewall Adjustment (if required)**
For **firewalld**-based systems (Grafana default port: **3000**):
```bash
sudo firewall-cmd --add-port=3000/tcp --permanent
sudo firewall-cmd --reload
```

## **Start Grafana**
- Navigate to Grafana directory:
  ```bash
  cd /home/observer/container/grafana
  ```
- Using Docker Compose v2:
  ```bash
  sudo docker compose up -d
  ```
- Or Legacy syntax:
  ```bash
  sudo docker-compose up -d
  ```

---

## **Verify Service is Running**
- Check container:
  ```bash
  sudo docker compose ps
  sudo docker ps --filter "name=grafana"
  ```
- View logs:
  ```bash
  sudo docker compose logs -f grafana
  # or
  sudo docker logs -f grafana
  ```
- Health test:
  ```bash
  curl -sSf http://192.168.0.106:3000/login
  ```
- Browser access:
  ```
  http://192.168.0.106:3000/
  ```
    Default login:
    - **Username:** admin
    - **Password:** admin
  
    **Note:** After first login, Grafana will prompt you to **change the password.**

---

## **Notes**
- **Default Port:** 3000
- **Grafana data directory:** `/home/observer/container/grafana/grafana_data`
- **Main configuration directory:** `/home/observer/container/grafana/grafana_config`
- **Logs directory:** `/home/observer/container/grafana/grafana_logs`
- If provisioning is empty → Grafana may show warnings.
  Create at least default layout:
  ```
  provisioning/
  ├── dashboards/
  ├── datasources/
  ```
  >**Note:** Or you can remove the provisioning mount entirely if not needed.
