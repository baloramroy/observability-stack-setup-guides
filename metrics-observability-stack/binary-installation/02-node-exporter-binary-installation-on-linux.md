---

# NODE EXPORTER BINARY INSTALLATION ON LINUX USING TARBALL

## OBJECTIVE

This SOP outlines the procedure for deploying **Node Exporter** on a Linux system using its **binary** distribution. Node Exporter is a **Prometheus exporter** that provides system metrics such as **CPU, memory, disk, and network** usage. The objective is Install node exporter (binary) on a Linux host so Prometheus server can **scrape** host metrics on port **9100**.

---

## DOWNLOAD THE NODE-EXPORTER TARBALL


### Download **Binary** package using command line:

Navigate to the **Prometheus Downloads** page and download the latest **Linux AMD64 tarball**.

`Or`

Download using this link (replace **X.Y.Z** with the version you choose):

  **Syntax**

    ```bash
    sudo wget \
    https://github.com/prometheus/node_exporter/releases/download/vX.Y.Z/node_exporter-X.Y.Z.linux-amd64.tar.gz
    ```
  
  **Example**
  
  - If the server has **internet** connection:
  
    ```bash
    sudo wget \
    https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
    ```
  
    > *Note: This will download the file to the current working directory.*
  
  - If the server has no internet **(using proxy)**:
  
    ```bash
    sudo wget -e use_proxy=yes \
    -e http_proxy=http://proxy_ip:port \
    -e https_proxy=http://proxy_ip:port \
    https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
    ```
  
    > *Note: This will download the file to the current working directory.*


### Extract and Rename Files

- Extract the downloaded package:

  ```bash
  sudo tar -xvf node_exporter-1.10.2.linux-amd64.tar.gz
  ```

- **Rename** the extracted directory to make future tasks easier:

  ```bash
  mv node_exporter-1.10.2.linux-amd64 node_exporter
  ```

---

## PREPARE THE SYSTEM

**Ensure system packages are updated**

- Debian/Ubuntu users:
  
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
  
- RHEL/CentOS/Fedora users:
  
  ```bash
  sudo yum update -y
  ```

---

## CREATE A DEDICATED USER AND GROUP

- Create node_exporter **user and group**:

  ```bash
  sudo useradd -M -s /usr/sbin/nologin node_exporter
  ```

  **Explanation:**
  
    * `-M` → Do not create home directory
    * `-s` → No login shell for the user

- Check if user/group was created:

  ```bash
  id node_exporter
  ```

  > *Output: User and group ID will be shown.*

---

## CREATE NECESSARY DIRECTORIES

- Node exporter does not need persistent storage, but optionally you can create:

  ```
  sudo mkdir -p /var/lib/node_exporter
  ```

---

## INSTALL THE BINARY FILES AND SET OWNERSHIP

**Purpose:** Move executables to a system path and set proper ownership.

- **Copy the binary file:**

  ```bash
  sudo cp node_exporter/node_exporter /usr/local/bin/
  ```

- **Set permissions of the binary file:**

  ```bash
  sudo chmod 0755 /usr/local/bin/node_exporter
  ```

- **Change ownership of the new binary file:**

  ```bash
  sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
  ```

- **Change ownership of data directory:**

  ```bash
  sudo chown -R node_exporter:node_exporter /var/lib/node_exporter
  ```

---

## OPEN FIREWALL PORTS (IF APPLICABLE)

**Purpose:** Allow remote access to Node Exporter on port **9100**.

- Allow mentioned port through firewall
  ```bash
  sudo firewall-cmd --add-port=9100/tcp --permanent
  ```
  
- Then Reload firewalld service daemon to take effect the change
  ```bash
  sudo firewall-cmd --reload
  ```

  > *Notes: Only needed if accessed remotely.*

---

## CREATE A SYSTEMD UNIT FILE TO RUN NODE EXPORTER AS A SERVICE

- **Create and open the service file:**
  ```bash
  sudo nano /etc/systemd/system/node_exporter.service
  ```

- **Paste the following:**

  ```
  [Unit]
  Description=Prometheus Node Exporter
  Wants=network-online.target
  After=network-online.target
  
  [Service]
  User=node_exporter
  Group=node_exporter
  Type=simple
  ExecStart=/usr/local/bin/node_exporter \
    --web.listen-address=0.0.0.0:9100 \
    --web.telemetry-path=/metrics \
    --collector.logind \
    --collector.processes \
    --collector.systemd \
    --collector.tcpstat
  
  # Custom log location
  #StandardOutput=append:/var/log/node_exporter/node_exporter.log
  #StandardError=append:/var/log/node_exporter/node_exporter-error.log
  
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```

- **Reload systemd and Start Node Exporter**

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable node_exporter
  sudo systemctl start node_exporter
  sudo systemctl status node_exporter
  ```

---

## VERIFY NODE EXPORTER METRICS

- Check in browser:
  ```
  http://<your-server-ip>:9100/metrics
  ```

- Or via curl:
  ```bash
  curl http://<your-server-ip>:9100/metrics
  ```

  > *Note: If successful, you’ll see CPU, memory, and filesystem metrics.*

---

## ACCESS NODE EXPORTER LOG

Node Exporter prints logs to **STDOUT/STDERR**, so systemd journal stores logs.

- View logs:

  ```bash
  sudo journalctl -u node_exporter
  sudo journalctl -u node_exporter -f
  sudo journalctl -u node_exporter --since "10 min ago"
  ```

---

## WRITE LOGS TO A FILE: OPTIONAL

Not recommended, but possible.

- Edit service file:
  ```bash
  sudo nano /etc/systemd/system/node_exporter.service
  ```

- Add below line:

  ```
  StandardOutput=append:/var/log/node_exporter/node_exporter.log
  StandardError=append:/var/log/node_exporter/node_exporter-error.log
  ```

- Create the **log** directory:

  ```bash
  sudo mkdir -p /var/log/node_exporter
  sudo chown node_exporter:node_exporter /var/log/node_exporter
  ```

- Reload and restart:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart node_exporter
  ```

---

## NOTES

* Default port: **9100**
* Binary path: `/usr/local/bin/node_exporter`
* Data path: `/var/lib/prometheus`
* Service file: `/etc/systemd/system/node_exporter.service`
* To view logs:
  ```
  sudo journalctl -u prometheus -f
  ```
* To view custom logs: `/var/log/prometheus`

---

## UNINSTALL / ROLLBACK

- Stop and disable service:
  ```bash
  sudo systemctl stop node_exporter && sudo systemctl disable node_exporter
  ```

- Remove service file:
  ```bash
  sudo rm -f /etc/systemd/system/node_exporter.service
  ```

- Reload systemd:
  ```bash
  sudo systemctl daemon-reload
  ```

- Remove binary:
  ```bash
  sudo rm -f /usr/local/bin/node_exporter
  ```

- Optional: remove user
  ```bash
  sudo userdel node_exporter
  ```

- Remove directories (if created)
  ```bash
  sudo rm -rf /var/lib/node_exporter /var/log/node_exporter
  ```

---
