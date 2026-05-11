# Alertmanager Binary Installation on Linux Using Tarball

## Objective

This SOP outlines the procedure for deploying **Alertmanager** on a Linux system using its **binary (tarball)** distribution. It includes steps to configure Alertmanager as a **systemd service**, prepare **directories**, and manage **logging and firewall access**.

---

## Download the Alertmanager tarball

1. Navigate to the **[Alertmanager Downloads page](https://prometheus.io/download/)** and download the latest **Linux AMD64 tarball**.

   `Or`

2. **Download Binary package using command line:**

   **Syntax:**

   * Download using this **link** (replace `X.Y.Z` with the **version** you choose):

     ```bash
     sudo wget https://github.com/prometheus/alertmanager/releases/download/vX.Y.Z/alertmanager-X.Y.Z.linux-amd64.tar.gz
     ```

   **Example:**

   * If the server has **internet connection**, download Alertmanager tarball using this command:

     ```bash
     sudo wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
     ```

     > **Note:** This will download the file to the current working directory

   * If the server has **no internet**, then use **proxy** to download Alertmanager tarball:

     ```bash
     sudo wget \
     -e use_proxy=yes \
     -e http_proxy=http://proxy_ip:port \
     -e https_proxy=http://proxy_ip:port \
     https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
     ```

     > **Note:** Replace the **proxy_ip:port** with the real IP and port

---

## Extract and rename files

* **Extract** the downloaded package:

  ```bash
  sudo tar -xvf alertmanager-0.27.0.linux-amd64.tar.gz
  ```

* **Rename** the extracted directory for future ease:

  ```bash
  mv alertmanager-0.27.0.linux-amd64 alertmanager
  ```

---

## Prepare The System

**Ensure system packages are updated:**

* For Debian / Ubuntu:

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

* For RHEL / CentOS / Rocky / Alma:

  ```bash
  sudo yum update -y
  ```

---

## Create a dedicated user and group

* Create **Alertmanager** user and group:

  ```bash
  sudo useradd -M -s /bin/false alertmanager
  ```

  * `-M` → Do not create home directory
  * `-s` → Disable shell login

* Verify user and group creation:

  ```bash
  id alertmanager
  ```

  **Output:** Displays UID and GID of alertmanager user.

---

## Create the necessary directories

* Directory to store **Alertmanager configuration**:

  ```bash
  sudo mkdir -p /etc/alertmanager
  ```

* Directory to store **Alertmanager runtime data**:

  ```bash
  sudo mkdir -p /var/lib/alertmanager
  ```

---

## Install the binaries and configuration files

**Purpose:** Move executables to system path and configuration to `/etc`.

### Copy the binary files

* Copy Alertmanager binary:

  ```bash
  sudo cp alertmanager/alertmanager /usr/local/bin/
  ```

* Copy amtool binary:

  ```bash
  sudo cp alertmanager/amtool /usr/local/bin/
  ```

  > **Note:** `amtool` is used to validate Alertmanager configuration.

#

### Copy the configuration file

* Copy default configuration file:

  ```bash
  sudo cp alertmanager/alertmanager.yml /etc/alertmanager/
  ```

---

## Set Proper Ownership and Permissions

* Set executable permissions:

  ```bash
  sudo chmod 0755 /usr/local/bin/alertmanager
  sudo chmod 0755 /usr/local/bin/amtool
  ```

* Change ownership of binaries:

  ```bash
  sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
  sudo chown alertmanager:alertmanager /usr/local/bin/amtool
  ```

* Change ownership of configuration directory:

  ```bash
  sudo chown -R alertmanager:alertmanager /etc/alertmanager
  ```

* Change ownership of data directory:

  ```bash
  sudo chown -R alertmanager:alertmanager /var/lib/alertmanager
  ```

---

## Validate Alertmanager Configuration

* Validate the configuration before starting the service:

  ```bash
  amtool check-config /etc/alertmanager/alertmanager.yml
  ```

  **Output:**
  `SUCCESS` if configuration is valid.

---

## Open firewall ports

**Purpose:** Allow remote access to Alertmanager Web UI (default port **9093**).

* Allow port in firewall:

  ```bash
  sudo firewall-cmd --add-port=9093/tcp --permanent
  ```

* Reload firewall rules:

  ```bash
  sudo firewall-cmd --reload
  ```

  > **Note:** Required only if Alertmanager UI is accessed remotely.

---

## Create a systemd unit file for Alertmanager

**Purpose:** Run Alertmanager as a background service and start on boot.

* Create service file:

  ```bash
  sudo nano /etc/systemd/system/alertmanager.service
  ```

* Paste the following content:

  ```ini
  [Unit]
  Description=Prometheus Alertmanager
  Wants=network-online.target
  After=network-online.target

  [Service]
  User=alertmanager
  Group=alertmanager
  Type=simple
  ExecStart=/usr/local/bin/alertmanager \
    --config.file=/etc/alertmanager/alertmanager.yml \
    --storage.path=/var/lib/alertmanager

  Restart=on-failure
  LimitNOFILE=65536

  [Install]
  WantedBy=multi-user.target
  ```

---

## Reload systemd and Start Alertmanager

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager
```

---

## Access Alertmanager Web UI

* Open browser:

  ```
  http://<your-server-ip>:9093
  ```

* Test using curl:

  ```bash
  curl http://<your-server-ip>:9093
  ```

---

## Access Alertmanager Logs

Alertmanager writes logs to **STDOUT/STDERR**, which are handled by **systemd journal**.

```bash
sudo journalctl -u alertmanager
sudo journalctl -u alertmanager -f
sudo journalctl -u alertmanager --since "10 min ago"
```

---

## Alertmanager Logs in a File (Optional, Not Recommended)

* Edit service file:

  ```bash
  sudo nano /etc/systemd/system/alertmanager.service
  ```

* Add the below lines in the service file:

  ```ini
  StandardOutput=append:/var/log/alertmanager/alertmanager.log
  StandardError=append:/var/log/alertmanager/alertmanager-error.log
  ```

* Create log directory:

  ```bash
  sudo mkdir -p /var/log/alertmanager
  sudo chown alertmanager:alertmanager /var/log/alertmanager
  ```

* Reload and restart:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart alertmanager
  ```

> **Note:** Using `journalctl` is the recommended approach.

---

## NOTES

* Default port: `9093`
* Configuration file: `/etc/alertmanager/alertmanager.yml`
* Binary path: `/usr/local/bin/alertmanager`
* Data path: `/var/lib/alertmanager`
* Service file: `/etc/systemd/system/alertmanager.service`
* Logs: `sudo journalctl -u alertmanager -f`

---

## UNINSTALL / ROLLBACK

* Stop and disable service:

  ```bash
  sudo systemctl stop alertmanager && sudo systemctl disable alertmanager
  ```

* Remove service file:

  ```bash
  sudo rm -f /etc/systemd/system/alertmanager.service
  ```

* Reload systemd:

  ```bash
  sudo systemctl daemon-reload
  ```

* Remove binaries:

  ```bash
  sudo rm -f /usr/local/bin/alertmanager /usr/local/bin/amtool
  ```

* Optional: remove user and directories:

  ```bash
  sudo userdel alertmanager
  sudo rm -rf /etc/alertmanager /var/lib/alertmanager
  ```

---
