# SOP: Deploy Uptime Kuma using Docker Compose

## 1. Purpose

To install and run **Uptime Kuma** as a containerized monitoring solution using **Docker Compose** for service uptime monitoring.

---

## 2. Scope

This SOP applies to Linux servers **(RHEL/CentOS)** with **Docker** installed, typically used in DevOps / Infrastructure **monitoring** environments.

---

## 3. Prerequisites

### 3.1 System Requirements

- OS: Linux (RHEL/CentOS 7/8/9)
- RAM: Minimum 1 GB (Recommended 2 GB)
- Disk: Minimum 5 GB free

### 3.2 Host OS with Docker Engine installed and running.

- Docker
- Docker Compose

### 3.3 Verify Installation

```bash
docker -v
docker compose version
```

---

## 4. Prepare the System

### 4.1 Create Dedicated User for the Service:

- Create **observer** user for this service
  ```bash
  sudo useradd observer
  ```
  
- Add **observer** user to **docker** group
  ```bash
  sudo usermod -aG docker observer
  ```
  
- Check **user and group** created or not
  ```bash
  id observer
  ```
  **Output:** It will output the user and group id with their name.

### 4.2 Create Network for Docker Container

- Here we will use a **custom bridge network** for **uprime kuma** docker container
  ```bash
  docker network create mon_net
  ```

---

## 5. Directory Structure and Permission

### 5.1 Directory Layout

```bash
/home/observer/pods/uptime-kuma/
├── docker-compose.yml
├── .env (optional)
└── uptimekuma_data/ (if using bind mount instead of volume)
```

#

### 5.2 Create Nessecary Directory for Uptime Kuma

- Create this directory if doesn't exist:

  ```bash
  mkdir -p /home/observer/pods/uptime-kuma
  ```
  
- Change to previously created `uptime-kuma` directory:
  
  ```bash
  cd /home/observer/pods/uptime-kuma
  ```
  
- Then create **persistent** data directory for **uptime kuma**:
  
  ```bash
  mkdir -p /home/observer/pods/uptime-kuma/uptimekuma_data
  ```
  > `uptimekuma_data` will be the data directory for the **uptime kuma** container.
#

### 5.3 Setup Permission

- Determine the container’s running user:
  ```bash
  docker run --rm -it --entrypoint /bin/sh elestio/uptime-kuma:latest
  / $ id
  uid=0(root) gid=0(root) groups=0(root)
  ```
  
- Assign correct ownership:
  ```bash
  sudo chown -R 0:0 uptimekuma_data
  ```

---

## 6. Docker Compose Configuration

### 6.1 Create a `docker-compose.yml` file 

- Run below command inside `/home/opsman/pods/uptime-kuma/` this folder:

  ```bash
  vim docker-compose.yml
  ```

### 6.2 Uptime kuma Compose File

```yaml
services:
  uptime-kuma:
    image: elestio/uptime-kuma:latest
    container_name: uptime-kuma
    restart: always

    ports:
      - "192.168.230.204:30001:3001"

    volumes:
      - ./uptimekuma_data:/app/data:z

    networks:
      - mon_net

    deploy:
      resources:
        limits:
          memory: 4096M
          cpus: "2"
          pids: 100

    security_opt:
      - no-new-privileges:true
      - label:type:container_t

networks:
  mon_net:
    external: true
```

---

## 7. Deployment Steps


- Start Container
  
  ```bash
  docker compose up -d
  ```
  
- Verify Running Status
  
  ```bash
  docker ps
  ```
  
  Expected:
  
  ```
  uptime-kuma   Up   0.0.0.0:3001->3001/tcp
  ```

---

## 8. Initial Setup

- Access via browser:

  ```
  http://<server-ip>:3001
  ```

- First-Time Configuration
  
  - Create **admin username**
  - Set strong **password**
  - Configure **timezone**

---

## 9. Data Storage Explanation

### **Uptime Kuma uses:**

- **SQLite DB (default)** stored inside:

  ```
  /app/data/kuma.db
  ```

- Persistent volume:

  ```
  uptimekuma_data
  ```

### **Note**

Recent versions include **embedded MariaDB-like behavior**, but still primarily rely on SQLite unless explicitly configured otherwise.

---

## 10. Common Operations

- Stop Service
  
  ```bash
  docker compose down
  ```
  
- Start Again
  
  ```bash
  docker compose up -d
  ```
  
- View Docker Logs
  
  ```bash
  docker compose logs -f
  #
  docker logs -f <container-name>
  ```

---

## 11. Backup Procedure

- Backup Volume

  ```bash
  docker run --rm \
    -v uptimekuma_data:/data \
    -v $(pwd):/backup \
    alpine tar czf /backup/uptime-kuma-backup.tar.gz /data
  ```

---


## 12. Troubleshooting

- Container Not Starting
  
  ```bash
  docker compose logs
  ```
  
- Port Conflict
  
  ```bash
  ss -tulnp | grep 3001
  ```
  
- Permission Issues
  
  ```bash
  chmod -R 755 /home/opsman/pods/uptime-kuma
  ```

---



