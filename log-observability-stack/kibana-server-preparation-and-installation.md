Perfect 👍
Now you are moving exactly in the **correct ELK learning order**.

Below is a **from-zero, step-by-step, detailed guide** to install **Kibana on a separate node** and connect it to your **existing 3-node Elasticsearch cluster**.

I’ll assume:
• Elasticsearch cluster is already **GREEN**
• Kibana will run on **a new VM (VM4)**
• OS: **CentOS Stream 9**

---

# Step 2: Install Kibana on a Separate Node (From Zero)

---

## 1. Lab Design (Kibana Node)

### 1.1 VM Plan

| VM  | Hostname    | IP (example)  | Role   |
| --- | ----------- | ------------- | ------ |
| VM4 | kibana-node | 192.168.10.20 | Kibana |

---

### 1.2 Minimum Resources (Kibana VM)

• CPU: 1 vCPU
• RAM: 2 GB (4 GB recommended)
• Disk: 20 GB

Kibana is **lightweight** compared to ES.

---

## 2. OS Preparation (Kibana Node)

---

### 2.1 Set Hostname

```bash
hostnamectl set-hostname kibana-node
reboot
```

---

### 2.2 Update System

```bash
dnf update -y
```

---

### 2.3 Configure `/etc/hosts` (VERY IMPORTANT)

Edit:

```bash
vi /etc/hosts
```

Add **all ES nodes + Kibana itself**:

```text
192.168.10.11  es-node-1
192.168.10.12  es-node-2
192.168.10.13  es-node-3
192.168.10.20  kibana-node
```

Why?
• Kibana talks to ES using hostnames
• Avoid DNS issues
• Stable connections

---

## 3. Install Kibana

---

### 3.1 Import Elastic GPG Key

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

---

### 3.2 Add Elastic Repository

Create repo file:

```bash
vi /etc/yum.repos.d/elastic.repo
```

Add:

```ini
[elastic]
name=Elastic repository
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

---

### 3.3 Install Kibana

```bash
dnf install kibana -y
```

⚠ Do NOT start Kibana yet.

---

## 4. Kibana Configuration (MOST IMPORTANT PART)

---

### 4.1 Main Configuration File

Edit:

```bash
vi /etc/kibana/kibana.yml
```

---

### 4.2 Basic Kibana Configuration

Add / modify the following:

```yaml
server.port: 5601
server.host: "0.0.0.0"

server.name: "kibana-node"

elasticsearch.hosts:
  - "http://es-node-1:9200"
  - "http://es-node-2:9200"
  - "http://es-node-3:9200"

kibana.index: ".kibana"
```

Explanation:
• `0.0.0.0` → allow browser access
• Multiple ES hosts → high availability
• Kibana auto-switches if one ES node fails

---

## 5. Firewall Configuration (Kibana Node)

Open Kibana port:

```bash
firewall-cmd --permanent --add-port=5601/tcp
firewall-cmd --reload
```

---

## 6. Start Kibana

---

### 6.1 Enable & Start Service

```bash
systemctl daemon-reexec
systemctl enable kibana
systemctl start kibana
```

---

### 6.2 Check Status

```bash
systemctl status kibana
```

Expected:

```text
Active: active (running)
```

If it fails:

```bash
journalctl -u kibana -f
```

---

## 7. Access Kibana UI

From your **browser**:

```
http://192.168.10.20:5601
```

You should see:
• Kibana loading screen
• Welcome page

If page doesn’t load:
• Check firewall
• Check Kibana logs
• Check ES cluster health

---

## 8. Verify Kibana ↔ Elasticsearch Connection

In Kibana UI:
• Go to **Stack Management**
• Elasticsearch should show **Connected**

Or via CLI on Kibana node:

```bash
curl http://es-node-1:9200
```

---

## 9. Common Problems & Fixes

### Kibana not starting

• ES cluster down
• Wrong ES hostnames
• Firewall blocked
• Memory too low

---

### Browser shows “Kibana server is not ready yet”

• ES not reachable
• ES still starting
• Check:

```bash
journalctl -u kibana -f
```

---

## 10. What You Have Achieved

Now you have:
• 3-node Elasticsearch cluster
• Dedicated Kibana node
• HA connection to ES
• Production-style separation

This is **exactly how real ELK environments are built**.

---

## Next Steps (Choose One)

1. Install **Logstash on separate node**
2. Enable **Elasticsearch & Kibana security**
3. Create **index patterns & dashboards**
4. Test **ES node failure from Kibana**
5. Understand **Kibana internals**

Tell me your next step and we’ll continue cleanly 👍
