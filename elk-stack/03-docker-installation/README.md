# Elastic Stack (Elasticsearch & Kibana) in Docker - SOP & Walkthrough

This repository contains a modular Docker setup for **Elasticsearch** and **Kibana**. Each service resides in its own directory with dedicated `docker-compose.yml`, configuration (`config/`), persistent storage (`data/`), and logs (`logs/`), communicating through a shared external Docker bridge network (`elastic-net`).

---

## 1. Directory Structure

```text
elastic-docker/
├── .gitignore
├── README.md                          <-- (This guide)
│
├── elasticsearch/
│   ├── docker-compose.yml             <-- Elasticsearch compose file
│   ├── .env                           <-- Active environment variables
│   ├── .env.example                   <-- Template environment variables
│   ├── config/
│   │   └── elasticsearch.yml          <-- Custom cluster & security settings
│   ├── data/                          <-- Persistent data mount (bind mount)
│   └── logs/                          <-- Elasticsearch log directory
│
└── kibana/
    ├── docker-compose.yml             <-- Kibana compose file
    ├── .env                           <-- Active environment variables
    ├── .env.example                   <-- Template environment variables
    ├── config/
    │   └── kibana.yml                 <-- Kibana configuration file
    └── data/                          <-- Persistent state/data mount
```

---

## 2. Prerequisites & System Configuration

### 2.1 Virtual Memory (`vm.max_map_count`)
Elasticsearch uses a `mmapfs` directory by default to store indices. The default operating system limits on mmap counts are too low.

- **Linux**:
  ```bash
  # Check current value
  sysctl vm.max_map_count

  # Set temporarily
  sudo sysctl -w vm.max_map_count=262144

  # Set permanently in /etc/sysctl.conf
  echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
  sudo sysctl -p
  ```

- **Windows (Docker Desktop with WSL2)**:
  Open PowerShell as Administrator:
  ```powershell
  wsl -d docker-desktop sysctl -w vm.max_map_count=262144
  ```
  *(Or add `vm.max_map_count=262144` to `%USERPROFILE%\.wslconfig` under `[wsl2]`)*.

### 2.2 Linux Host Permissions (Crucial for Linux Deployments)
Elasticsearch runs inside the container as user `elasticsearch` (UID `1000`, GID `0` or `1000`). If deploying on Linux, set ownership for mounted data directories:
```bash
sudo chown -R 1000:0 elasticsearch/data elasticsearch/logs
sudo chmod -R 775 elasticsearch/data elasticsearch/logs

sudo chown -R 1000:0 kibana/data
```
*(On Windows / Docker Desktop, file permissions are automatically mapped via the virtualization layer).*

---

## 3. Step-by-Step Setup Walkthrough

### Step 1: Create the External Docker Network
Because Elasticsearch and Kibana have independent compose files in separate directories, they need a common bridge network to discover each other by container name.

Run once:
```bash
docker network create elastic-net
```

Verify:
```bash
docker network ls
```

---

### Step 2: Configure & Start Elasticsearch

1. Navigate to the `elasticsearch` directory:
   ```bash
   cd elasticsearch
   ```

2. Inspect/edit `.env` (adjust heap memory or ports if needed):
   ```ini
   STACK_VERSION=8.17.2
   ES_PORT=9200
   ES_JAVA_OPTS=-Xms1g -Xmx1g
   ```
   > **Note on Heap Memory**: For production, allocate 50% of available RAM to Elasticsearch heap (e.g. `-Xms4g -Xmx4g` for an 8GB node), leaving the remaining 50% for OS filesystem cache.

3. Start Elasticsearch:
   ```bash
   docker compose up -d
   ```

4. Monitor the startup logs:
   ```bash
   docker compose logs -f elasticsearch
   ```

5. Verify health:
   ```bash
   curl -X GET "http://localhost:9200/_cluster/health?pretty"
   ```
   Or open `http://localhost:9200` in your browser. You should receive a JSON response with status `"green"` or `"yellow"`.

---

### Step 3: Configure & Start Kibana

1. Open a new terminal or navigate to the `kibana` directory:
   ```bash
   cd ../kibana
   ```

2. Inspect `.env`:
   ```ini
   STACK_VERSION=8.17.2
   KIBANA_PORT=5601
   ```

3. Start Kibana:
   ```bash
   docker compose up -d
   ```

4. Monitor logs until Kibana reports that the server is ready:
   ```bash
   docker compose logs -f kibana
   ```

5. Access Kibana Web UI:
   Open **[http://localhost:5601](http://localhost:5601)** in your browser.

---

## 4. Security Configuration (Two Operating Modes)

### Mode A: Development / Internal Mode (Default)
In `elasticsearch/config/elasticsearch.yml`:
```yaml
xpack.security.enabled: false
```
- **Pros**: Zero initial setup friction; no tokens, certificates, or passwords needed.
- **Use Case**: Local development, testing, or internal isolated networks behind an authenticating reverse proxy (like Nginx/Traefik).

---

### Mode B: Production / Secured Mode (With Authentication)
To enforce user authentication and role-based access control:

1. In `elasticsearch/config/elasticsearch.yml`, enable security:
   ```yaml
   xpack.security.enabled: true
   xpack.security.http.ssl.enabled: false
   xpack.security.transport.ssl.enabled: false
   ```

2. In `elasticsearch/docker-compose.yml`, pass an initial superuser password under `environment`:
   ```yaml
   - ELASTIC_PASSWORD=YourStrongPasswordHere
   ```

3. Restart Elasticsearch:
   ```bash
   docker compose down
   docker compose up -d
   ```

4. Generate a Service Account Token for Kibana (recommended for 8.x):
   ```bash
   docker exec -it elasticsearch bin/elasticsearch-service-tokens create elastic/kibana kibana-token
   ```
   *Copy the generated token string.*

5. Configure Kibana with the token:
   In `kibana/config/kibana.yml`:
   ```yaml
   elasticsearch.serviceAccountToken: "AAEAAWVsYXN0aWMva2liYW5hL2tpYmFuYS10b2tlbj..."
   ```

6. Restart Kibana:
   ```bash
   cd ../kibana
   docker compose restart
   ```

Now you can log into `http://localhost:5601` using:
- **Username**: `elastic`
- **Password**: `YourStrongPasswordHere`

---

## 5. Day-2 Operations & Management

### Managing Services
| Action | Elasticsearch Directory | Kibana Directory |
| :--- | :--- | :--- |
| **Start** | `docker compose up -d` | `docker compose up -d` |
| **Stop** | `docker compose stop` | `docker compose stop` |
| **Restart** | `docker compose restart` | `docker compose restart` |
| **View Logs** | `docker compose logs -f` | `docker compose logs -f` |
| **Check Status** | `docker compose ps` | `docker compose ps` |
| **Tear Down** | `docker compose down` | `docker compose down` |

### Data Persistence
- **Elasticsearch data**: Stored in `./elasticsearch/data/`. All indices, shards, and settings remain intact even if containers are destroyed.
- **Kibana state**: Stored in `./kibana/data/` and within the `.kibana` index inside Elasticsearch.

---

## 6. Troubleshooting Guide

| Issue | Root Cause | Solution |
| :--- | :--- | :--- |
| `max virtual memory areas vm.max_map_count [65530] is too low` | Linux kernel memory map limit is below the Elasticsearch minimum (262144). | Run `sudo sysctl -w vm.max_map_count=262144` or in WSL `wsl -d docker-desktop sysctl -w vm.max_map_count=262144`. |
| `AccessDeniedException: /usr/share/elasticsearch/data/nodes` | Linux directory permissions issue on host `./data` mount. | Run `sudo chown -R 1000:0 elasticsearch/data elasticsearch/logs`. |
| `network elastic-net declared as external, but could not be found` | The shared Docker network was not created prior to starting compose. | Run `docker network create elastic-net`. |
| Kibana shows `"Kibana server is not ready yet"` | Elasticsearch is still initializing, or network connection between containers failed. | Check Elasticsearch health via `curl http://localhost:9200/_cluster/health`. Ensure both containers are attached to `elastic-net`. |
| Container killed with exit code 137 | Out of Memory (OOM). | Adjust `ES_JAVA_OPTS` in `elasticsearch/.env` to lower heap or give Docker more RAM. |
