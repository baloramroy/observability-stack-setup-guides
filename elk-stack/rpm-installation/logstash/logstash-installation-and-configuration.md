Excellent 👍
You’re now completing the **core ELK architecture** the *right* way.

Below is a **from-zero, step-by-step, detailed guide** to install **Logstash on a separate node** and connect it to your **existing Elasticsearch cluster**.

Assumptions:
• Elasticsearch 3-node cluster is **GREEN**
• Kibana is already running on its own node
• OS: **CentOS Stream 9**
• Logstash will be used as **central log processor**

---

# Step 3: Install Logstash on a Separate Node (From Zero)

---

## 1. Lab Design (Logstash Node)

### 1.1 VM Plan

| VM  | Hostname      | IP (example)  | Role     |
| --- | ------------- | ------------- | -------- |
| VM5 | logstash-node | 192.168.10.30 | Logstash |

---

### 1.2 Minimum Resources (Logstash VM)

• CPU: 2 vCPU
• RAM: 2–4 GB
• Disk: 20–30 GB

Logstash is **CPU intensive**, especially with grok filters.

---

## 2. OS Preparation (Logstash Node)

---

### 2.1 Set Hostname

```bash
hostnamectl set-hostname logstash-node
reboot
```

---

### 2.2 Update System

```bash
dnf update -y
```

---

### 2.3 Configure `/etc/hosts` (CRITICAL)

Edit:

```bash
vi /etc/hosts
```

Add **all ELK nodes**:

```text
192.168.10.11  es-node-1
192.168.10.12  es-node-2
192.168.10.13  es-node-3
192.168.10.20  kibana-node
192.168.10.30  logstash-node
```

Why?
• Logstash sends data to ES
• Uses hostnames for failover
• Avoid DNS problems

---

## 3. Install Logstash

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

### 3.3 Install Logstash

```bash
dnf install logstash -y
```

⚠ Do NOT start Logstash yet.

---

## 4. Logstash Basics (Important for Understanding)

Before configuring, understand **Logstash pipeline**:

```
INPUT → FILTER → OUTPUT
```

• Input: where logs come from
• Filter: parse/enrich logs
• Output: send to Elasticsearch

---

## 5. Create First Logstash Pipeline (Learning-Friendly)

We will start with:
✔ Simple input
✔ No complex grok
✔ Clean verification

---

### 5.1 Create Pipeline Directory (if not exists)

```bash
mkdir -p /etc/logstash/conf.d
```

---

### 5.2 Create Test Pipeline

```bash
vi /etc/logstash/conf.d/test.conf
```

Add:

```conf
input {
  stdin {
  }
}

filter {
}

output {
  elasticsearch {
    hosts => ["http://es-node-1:9200","http://es-node-2:9200","http://es-node-3:9200"]
    index => "logstash-test-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

Explanation:
• `stdin` → manual input for testing
• `stdout` → see parsed output
• ES output → sends data to cluster
• Multiple hosts → HA

---

## 6. JVM Heap Configuration (IMPORTANT)

Edit:

```bash
vi /etc/logstash/jvm.options
```

Set:

```text
-Xms1g
-Xmx1g
```

Rule:
• Heap = ~50% RAM
• Min = Max

---

## 7. Firewall Configuration (If Using Beats Later)

Open Beats input port (optional now):

```bash
firewall-cmd --permanent --add-port=5044/tcp
firewall-cmd --reload
```

(5044 is default Beats → Logstash port)

---

## 8. Test Logstash Configuration (VERY IMPORTANT)

Before starting service, **always test config**:

```bash
/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Expected:

```text
Configuration OK
```

If errors:
• Syntax issue
• Wrong plugin config

---

## 9. Start Logstash

---

### 9.1 Enable & Start

```bash
systemctl daemon-reexec
systemctl enable logstash
systemctl start logstash
```

---

### 9.2 Check Status

```bash
systemctl status logstash
```

Must be:

```text
Active: active (running)
```

Logs:

```bash
journalctl -u logstash -f
```

---

## 10. Test Logstash End-to-End (IMPORTANT)

---

### 10.1 Run Logstash in Foreground (One-Time Test)

Stop service temporarily:

```bash
systemctl stop logstash
```

Run manually:

```bash
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/test.conf
```

---

### 10.2 Send Test Message

Type:

```text
Hello ELK from Logstash
```

You should see:
• Parsed output in terminal
• No errors

Stop with:

```text
CTRL + C
```

---

### 10.3 Verify in Elasticsearch

From any ES node:

```bash
curl http://localhost:9200/_cat/indices?v
```

You should see:

```text
logstash-test-YYYY.MM.dd
```

---

## 11. Verify in Kibana

In Kibana UI:
• Stack Management → Index Patterns
• Create pattern:

```text
logstash-test-*
```

• Time field: `@timestamp`
• Discover → See your message

---

## 12. Common Beginner Mistakes

• Starting Logstash without pipeline
• Not testing config (`-t`)
• Wrong ES hostnames
• Low memory → Logstash killed
• Grok before basics

---

## 13. What You Have Achieved

Now you have:
✔ 3-node Elasticsearch cluster
✔ Separate Kibana node
✔ Separate Logstash node
✔ End-to-end working ELK stack

This is **real production-style ELK**, not demo setup.

---

## Next Steps (Highly Recommended)

Choose what you want next:

1. Send **real system logs** via Filebeat
2. Learn **Logstash grok filters (deep dive)**
3. Enable **ELK security (users, TLS)**
4. Index lifecycle management (ILM)
5. ES node roles (master/data/ingest)

Tell me your next step — we’ll go deeper 💪
