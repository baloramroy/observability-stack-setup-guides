# Elasticsearch Installation Prerequisites & Configurations

Before installing **Elasticsearch** on the host machine, we need to take care of **certain prerequisites** and **configurations**. Let's go through them one by one.

## 1. Lab Design

### 1.1 VM Plan

| VM  | Hostname  | IP (example)  | Role                   |
| --- | --------- | ------------- | ---------------------- |
| VM1 | es-node-1 | 192.168.10.11 | Master + Data + Ingest |
| VM2 | es-node-2 | 192.168.10.12 | Master + Data + Ingest |
| VM3 | es-node-3 | 192.168.10.13 | Master + Data + Ingest |

>Use **static IPs**. Elasticsearch hates changing IPs.

#

### 1.2 Minimum Resources (per ES node)

-  CPU: 2 vCPU
-  RAM: **4 GB minimum** (8 GB recommended)
-  Disk: 40 GB
-  Network: Same subnet

---

## 2. OS-Level Preparation

Do these steps **on ALL 3 nodes**.

#

### 2.1 Set Hostname

- On es-node-1

  ```bash
  hostnamectl set-hostname es-node-1
  ```

- On es-node-2

  ```bash
  hostnamectl set-hostname es-node-2
  ```

- On es-node-3

  ```bash
  hostnamectl set-hostname es-node-3
  ```

- Reboot all nodes:

  ```bash
  reboot
  ```

---

### 2.2 Update System

- This is not mandatory

  ```bash
  dnf update -y
  ```

---

### 2.3 Configure `/etc/hosts` (CRITICAL)

- Edit on **ALL nodes**:

  ```bash
  vim /etc/hosts
  ```

- Add hostname in system hosts file:

  ```text
  192.168.10.11  es-node-1
  192.168.10.12  es-node-2
  192.168.10.13  es-node-3
  ```

Why need hostname entry in the hostfile?
- ES discovery uses hostnames
- Prevents DNS issues
- Faster cluster formation

---

### 2.4 Disable Swap (MANDATORY)

Elasticsearch **will not work properly with swap**.

- Disable swap on runtime

  ```bash
  swapoff -a
  ```

- Disable swap permanently:

  ```bash
  vim /etc/fstab
  ```

- Comment out swap line:

  ```text
  # /dev/mapper/cs-swap swap swap defaults 0 0
  ```

- Verify disable or not:

  ```bash
  free -h
  ```

  >Swap must be `0`.

---

### 2.5 Increase virtual memory

On Linux, you can increase the limits of the `vm.max_map_count` parameter by following this step:

Think:\
**vm.max_map_count = how many "file-to-memory mappings" Linux allows**

#

- Create config file:

  ```bash
  vim /etc/sysctl.d/99-elasticsearch.conf
  ```

- Add below line:

  ```text
  # Recommended (Official Elasticsearch requirement)
  vm.max_map_count=262144

  # Modern ES Value (Not needed for small setup)
  vm.max_map_count=1048576 
  ```

- Apply:

  ```bash
  sysctl --system
  ```

- Verify:

  ```bash
  sysctl vm.max_map_count
  ```

  Expected:

  ```text
  vm.max_map_count = 262144
  ```

---

### 2.6 File Descriptor Limits

There are **3 different** ways of apply **file descriptor limits** in Linux. But remember,
- If ES started from terminal like `./bin/elasticsearch` → uses `ulimit` or `limits.conf` to apply limit.
- If started by systemd → uses systemd limits

As a prerequites meetup, we setting **file descriptor limits** by using `limits.conf` option. So that if someone run elasticsearch from **shell** this limits get applied.
#

- Create and Edit below file:

  ```bash
  vim /etc/security/limits.d/elasticsearch.conf
  ```

- Add these lines:

  ```text
  elasticsearch soft nofile 65535
  elasticsearch hard nofile 65535
  elasticsearch soft nproc  4096
  elasticsearch hard nproc  4096
  ```

- After starting ES check this:

  ```bash
  cat /proc/$(pidof java)/limits | grep "Max open files"
  ```

**NOTE:**\
This does NOT apply when **Elasticsearch** runs as a **systemd service**.
We will configure **systemd limits** later.

---

### 2.7 Disable Transparent Huge Pages (THP)

This is **very important for Elasticsearch performance**.

#

**What is THP?**

Transparent Huge Pages tries to optimize memory by using large pages automatically.

Sounds good — but for Elasticsearch:
- Causes latency spikes
- Unpredictable GC pauses
- Memory fragmentation issues



**Check THP status:**

- Run this to check:

  ```bash
  cat /sys/kernel/mm/transparent_hugepage/enabled
  ```

- If you see:

  ```text
  [always] madvise never
  ```

  → BAD (enabled)

#

**Disable THP (Temporary):**
- Run Below Command:

  ```bash
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
  ```

#

**Disable THP (Permanent)**

- Create a systemd service:

  ```bash
  vim /etc/systemd/system/disable-thp.service
  ```

- Add:

  ```ini
  [Unit]
  Description=Disable Transparent Huge Pages
  After=network.target

  [Service]
  Type=oneshot
  ExecStart=/usr/bin/bash -c "echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag"

  [Install]
  WantedBy=multi-user.target
  ```

- Enable it:

  ```bash
  systemctl daemon-reload
  systemctl enable disable-thp
  systemctl start disable-thp
  ```

---

## Firewall Configuration

Required Ports

| Port | Purpose |
|----|--------|
| 22 | SSH |
| 9200 | ES HTTP |
| 9300 | ES Transport |

Run bwlow comman to configure firewall:

```bash
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --permanent --add-port=9300/tcp
firewall-cmd --reload
```

---

## Network Connectivity Check

Check connectivity between dodes:
```bash
ping es-node-2
nc -zv es-node-2 9300
```

ES cluster fails silently if network blocked

## Time Synchronization

```bash
timedatectl set-ntp true
timedatectl status
```
Why important:
- Cluster coordination depends on time
- Prevents split-brain / election issues
  
---

## Final Pre-Flight Checklist

- **IP Address** – `ip a"`

- **Hostname** – `hostname`

- **Hosts File Entries** – `cat /etc/hosts | grep es-node`

- **Swap Status** – `free -h | grep Swap`

- **Kernel Parameter** – `sysctl vm.max_map_count`

- **Transparent Huge Pages (THP)** – `cat /sys/kernel/mm/transparent_hugepage/enabled`

- **File Descriptor Limits** – `cat /etc/security/limits.d/elasticsearch.conf`

- **Firewall Ports** – `firewall-cmd --list-ports | grep -E "9200|9300"`

- **Time Synchronization** – `timedatectl status | grep -E "NTP service|System clock"`

---
