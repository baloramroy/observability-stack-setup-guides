
# Grafana Binary Installation On Linux Using Tarball

## Objective

Grafana is a **Prometheus** visualization tool. This SOP outlines the procedure for **deploying** Grafana on a Linux system using its **binary** distribution. It includes instructions for **configuring Grafana** server from official binary `.tar.gz` on a Linux host, running it as a systemd service on port **3000**, and configure basic security/permission settings.

---

## Download The Grafana Official Tarball

> **Note:** Navigate to the [Grafana Downloads](https://grafana.com/grafana/download?utm_source=chatgpt.com) page and download the latest **(Standalone Linux Binaries)** Linux AMD64 tarball.

### Download Binary package using command line:

- Suppose we are working at our current **home directory** which is:

  ```
  /home/broy/download
  ```

- If the server has **internet connection** then download Grafana tarball using this command:
  
  ```bash
  sudo wget https://dl.grafana.com/grafana/release/12.3.0/grafana_12.3.0_19497075765_linux_amd64.tar.gz
  ```
  
  > **Note:** This will download the file to the current working directory.


- If the server has **no internet**, then use **proxy** to download Grafana tarball using this command:

  ```bash
  sudo wget -e use_proxy=yes \
  -e http_proxy=http://proxy_ip:port \
  -e https_proxy=http://proxy_ip:port \
  https://dl.grafana.com/grafana/release/12.3.0/grafana_12.3.0_19497075765_linux_amd64.tar.gz
  ```

  > **Note:** Replace **proxy ip** with **real proxy ip**.


### Extract files

- Go to the download directory
  
  ```bash
  cd /home/broy/download
  ```
  > **Output:** This will take you to the download directory.

- Extract the downloaded package:
  
  ```bash
  sudo tar -zxvf grafana_12.3.0_19497075765_linux_amd64.tar.gz
  ```
  
  > **Output:** There will be an extracted folder named `grafana_12.3.0`.
  

---

## Prepare The System

### Ensure system packages are updated

- Debian/Ubuntu:

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- RHEL/CentOS/Fedora:

  ```bash
  sudo yum update -y
  ```


### Create a dedicated user and group

- Create grafana user and group:

  ```bash
  sudo useradd -M -s /usr/sbin/nologin grafana
  ```
  
  * `-M`, `--no-create-home` → Do not create home directory
  * `-s`, `--shell` → No login shell for the user

- Check user and group:

  ```bash
  id grafana
  ```

  > **Output:** It will output the user and group id with their name.


### Create the necessary directories

- First create Grafana’s configuration directory:

  ```bash
  sudo mkdir -p /etc/grafana
  ```

- Then create Grafana’s main persistent storage directory:

  ```bash
  sudo mkdir -p /var/lib/grafana
  ```

- Then create Grafana’s main Logs directory:

  ```bash
  sudo mkdir -p /var/log/grafana
  ```

**All in one Command:**

```bash
sudo mkdir -p /etc/grafana /var/lib/grafana /var/log/grafana
```

---

## Install The Binaries Files And Set Ownership

**Purpose:** Move executables to a **system path** and set **proper ownership** for the new file and directories.

### Copy the Extracted file and folder to the installation path

- From the downloaded folder copy `grafana_12.3.0` folder to the `/usr/local` folder:

  ```bash
  sudo cp -r downloads/grafana_12.3.0 /usr/local/grafana
  ```

- Copy the **sample.ini** config file to a **persistent location**

  ```bash
  sudo cp /usr/local/grafana/conf/sample.ini /etc/grafana/grafana.ini
  ```

**Note:**
> In the grafana config directory there is a `default.ini` config file for Grafana. But Grafana docs recommend editing `custom.ini`, never `defaults.ini`.
So from the grafana directory we copy the provided `sample.ini` file and rename it `grafana.ini` config file and then paste it to the `/etc` config directory.


### Set Proper Ownership to the created directory

- Change the owner of `/usr/local/grafana` to grafana user:

  ```bash
  sudo chown grafana:grafana /usr/local/grafana/
  ```

- Change ownership of Grafana’s configuration directory:

  ```bash
  sudo chown -R grafana:grafana /etc/grafana
  ```

- Change ownership of Grafana’s main persistent storage directory:

  ```bash
  sudo chown -R grafana:grafana /var/lib/grafana
  ```

- Change ownership of Grafana’s main logs directory:

  ```bash
  sudo chown -R grafana:grafana /var/log/grafana
  ```

**All in one command:**

```bash
sudo chown -R grafana:grafana /usr/local/grafana /etc/grafana /var/lib/grafana /var/log/grafana
```

---

## Open Firewall Ports (If Applicable)

**Purpose:** Allow remote access to Grafana web UI. The default port is **3000**.

- Allow this port:

  ```bash
  sudo firewall-cmd --add-port=3000/tcp --permanent
  ```

- Reload firewall:

  ```bash
  sudo firewall-cmd --reload
  ```

  > **Notes:** This permission is only needed if this service needs to be accessed by others **remotely**.

---

## Create A Systemd Unit File To Run Grafana As A Service

- Create and open this file:

  ```bash
  nano /etc/systemd/system/grafana-server.service
  ```

- Paste the following content:

  ```ini
  [Unit]
  Description=Grafana Visualizer
  Wants=network.target
  After=network.target
  
  [Service]
  Type=simple
  User=grafana
  Group=grafana
  
  Environment=GF_PATHS_CONFIG=/etc/grafana/grafana.ini
  Environment=GF_PATHS_DATA=/var/lib/grafana
  Environment=GF_PATHS_LOGS=/var/log/grafana
  WorkingDirectory=/usr/local/grafana

  ExecStart=/usr/local/grafana/bin/grafana-server \
  --config="${GF_PATHS_CONFIG}"\
  --homepath=/usr/local/grafana

  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```

---

## Reload systemd and Start Grafana

```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

---

## Verify Grafana

**Open your browser or use curl to check**

```
http://<your-server-ip>:3000
```

Default Login:
- **Username:** admin
- **Password:** admin

**Note:** Change password on first login

---

## Access Grafana Log

### Access using journalctl

- You view them using the journalctl logging system:

  ```bash
  sudo journalctl -u grafana-server
  sudo journalctl -u grafana-server -f
  sudo journalctl -u grafana-server --since "10 min ago"
  ```

### Access custom logs

- Follow logs in real time:

  ```bash
  sudo tail -f /var/log/grafana/grafana.log
  ```

---

# Notes

* Default port: **3000**
* Binary path: `/usr/local/bin/grafana/grafana-server`
* Configuration file: `/etc/grafana/grafana.ini`
* Data path: `/var/lib/grafana`
* Service file: `/etc/systemd/system/grafana-server.service`
* To view logs: `sudo journalctl -u prometheus -f`
* To view custom logs: `/var/log/grafana`

---

## Uninstall / Rollback

- Stop service:

  ```bash
  sudo systemctl stop grafana-server && sudo systemctl disable grafana-server
  ```

- Remove service file:

  ```bash
  sudo rm -f /etc/systemd/system/grafana-server.service
  ```

- Reload daemon:

  ```bash
  sudo systemctl daemon-reload
  ```

- Remove all directories:

  ```bash
  sudo rm -rf /usr/local/grafana /etc/grafana /var/lib/grafana /var/log/grafana
  ```

- Optionally remove user:

  ```bash
  sudo userdel node_exporter
  ```

---

