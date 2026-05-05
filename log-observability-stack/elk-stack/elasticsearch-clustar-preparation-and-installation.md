# Step 1: Elasticsearch 3-Node Cluster Installation
---

## 1. Lab Design

### 1.1 VM Plan

| VM  | Hostname  | IP (example)  | Role          |
| --- | --------- | ------------- | ------------- |
| VM1 | es-node-1 | 192.168.10.11 | Elasticsearch |
| VM2 | es-node-2 | 192.168.10.12 | Elasticsearch |
| VM3 | es-node-3 | 192.168.10.13 | Elasticsearch |

⚠ Use **static IPs**. Elasticsearch hates changing IPs.

---

### 1.2 Minimum Resources (per ES node)

• CPU: 2 vCPU
• RAM: **4 GB minimum** (8 GB recommended)
• Disk: 40 GB
• Network: Same subnet

---

## 2. OS-Level Preparation

Do these steps **on ALL 3 nodes**.


### 2.1 Set Hostname

**On es-node-1**

```bash
hostnamectl set-hostname es-node-1
```

**On es-node-2**

```bash
hostnamectl set-hostname es-node-2
```

**On es-node-3**

```bash
hostnamectl set-hostname es-node-3
```

**Reboot all nodes:**

```bash
reboot
```

---

### 2.2 Update System

```bash
dnf update -y
```

---

### 2.3 Configure `/etc/hosts` (CRITICAL)

Edit on **ALL nodes**:

```bash
vi /etc/hosts
```

Add hostname in system hosts file:

```text
192.168.10.11  es-node-1
192.168.10.12  es-node-2
192.168.10.13  es-node-3
```

Why?
- ES discovery uses hostnames
- Prevents DNS issues
- Faster cluster formation

---

### 2.4 Disable Swap (MANDATORY)

Elasticsearch **will not work properly with swap**.

```bash
swapoff -a
```

Disable permanently:

```bash
vi /etc/fstab
```

Comment out swap line:

```text
# /dev/mapper/cs-swap swap swap defaults 0 0
```

Verify:

```bash
free -h
```

Swap must be `0`.

---

### 2.5 Kernel & System Limits (VERY IMPORTANT)

Create config file:

```bash
vi /etc/sysctl.d/99-elasticsearch.conf
```

Add:

```text
vm.max_map_count=262144
```

Apply:

```bash
sysctl -p /etc/sysctl.d/99-elasticsearch.conf
```

Verify:

```bash
sysctl vm.max_map_count
```

---

### 2.6 File Descriptor Limits

Edit:

```bash
vi /etc/security/limits.conf
```

Add at bottom:

```text
elasticsearch soft nofile 65535
elasticsearch hard nofile 65535
elasticsearch soft nproc  4096
elasticsearch hard nproc  4096
```

---

## 3. Install Elasticsearch (All Nodes)

---

### 3.1 Import Elastic GPG Key

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

---

### 3.2 Add Elasticsearch Repository

```bash
vi /etc/yum.repos.d/elasticsearch.repo
```

Add:

```ini
[elasticsearch]
name=Elasticsearch repository
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

---

### 3.3 Install Elasticsearch

```bash
dnf install elasticsearch -y
```

⚠ Do NOT start it yet.

---

## 4. Elasticsearch Configuration (MOST IMPORTANT PART)

This is where clusters are made or broken.

---

### 4.1 Main Config File

Edit:

```bash
vi /etc/elasticsearch/elasticsearch.yml
```

---

### 4.2 Configuration for **es-node-1**

```yaml
cluster.name: elk-lab-cluster

node.name: es-node-1

network.host: 192.168.10.11
http.port: 9200

discovery.seed_hosts:
  - es-node-1
  - es-node-2
  - es-node-3

cluster.initial_master_nodes:
  - es-node-1
  - es-node-2
  - es-node-3

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
```

---

### 4.3 Configuration for **es-node-2**

Change only these:

```yaml
node.name: es-node-2
network.host: 192.168.10.12
```

Everything else same.

---

### 4.4 Configuration for **es-node-3**

Change:

```yaml
node.name: es-node-3
network.host: 192.168.10.13
```

---

## 5. JVM Heap Configuration (CRITICAL)

Edit:

```bash
vi /etc/elasticsearch/jvm.options.d/heap.options
```

Add:

```text
-Xms2g
-Xmx2g
```

Rules:
• Heap = 50% of RAM
• Max heap = 32 GB
• Min = Max

---

## 6. Firewall Configuration

Open required ports on **ALL nodes**:

```bash
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --permanent --add-port=9300/tcp
firewall-cmd --reload
```

Ports:
• 9200 → REST API
• 9300 → Cluster communication

---

## 7. Start Elasticsearch Cluster

### 7.1 Enable Service

```bash
systemctl daemon-reexec
systemctl enable elasticsearch
```

---

### 7.2 Start Nodes (ORDER MATTERS FIRST TIME)

Start **one by one**, not all together.

#### On es-node-1

```bash
systemctl start elasticsearch
```

Wait 1–2 minutes.

#### On es-node-2

```bash
systemctl start elasticsearch
```

Wait.

#### On es-node-3

```bash
systemctl start elasticsearch
```

---

### 7.3 Check Service Status

```bash
systemctl status elasticsearch
```

Must be:

```text
Active: active (running)
```

---

## 8. Verify Cluster (VERY IMPORTANT)

From **any node**:

---

### 8.1 Check Cluster Health

```bash
curl -k http://localhost:9200/_cluster/health?pretty
```

Expected:

```json
"status" : "green",
"number_of_nodes" : 3
```

---

### 8.2 Check Nodes

```bash
curl -k http://localhost:9200/_cat/nodes?v
```

You should see:

```text
es-node-1
es-node-2
es-node-3
```

---

### 8.3 Check Cluster Name

```bash
curl -k http://localhost:9200
```

---

## 9. Common Beginner Mistakes (READ THIS)

• Forgetting swapoff
• Wrong IP in `network.host`
• Firewall not opened
• Heap size too large
• Starting all nodes at once first time
• Hostname not resolvable

---

## 10. What You Have Achieved

At this point:
• 3-node ES cluster
• Proper quorum
• Production-style setup
• Ready for Kibana & Logstash

This is **REAL Elasticsearch**, not demo junk.

---

## Next Steps (Choose One)

Tell me what you want next:

1. Install **Kibana on separate node**
2. Install **Logstash on separate node**
3. Explain **ES master / data / ingest roles**
4. Test **node failure & recovery**
5. Enable **security (users, passwords, TLS)**

We’ll go **one clean step at a time**.
