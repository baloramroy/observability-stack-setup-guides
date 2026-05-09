Excellent choice 👍
**Filebeat is where ELK becomes “real”**, because now you’ll ingest **actual system logs**, not test data.

Below is a **from-zero, step-by-step, detailed guide** to send **real Linux system logs via Filebeat → Logstash → Elasticsearch → Kibana**.

This follows **production-style architecture**.

---

# Step 4: Send Real System Logs via Filebeat (From Zero)

---

## 0. Architecture (Understand First)

We will build this flow:

```
[Linux Server]
   |
   |  (Filebeat)
   v
[Logstash]
   |
   v
[Elasticsearch Cluster]
   |
   v
[Kibana]
```

For learning, we’ll install **Filebeat on the Logstash node itself**
(later you can install it on any app/server).

---

## 1. Decide Log Source (Learning-Friendly)

We’ll start with **real system logs**:

• `/var/log/messages`
• `/var/log/secure`

These logs:
✔ Always exist
✔ Contain real events
✔ Easy to understand

---

## 2. Install Filebeat (On Logstash Node)

### 2.1 Login to `logstash-node`

```bash
ssh root@logstash-node
```

---

### 2.2 Import Elastic GPG Key

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

---

### 2.3 Add Elastic Repository (If Not Exists)

```bash
vi /etc/yum.repos.d/elastic.repo
```

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

### 2.4 Install Filebeat

```bash
dnf install filebeat -y
```

---

## 3. Configure Filebeat (MOST IMPORTANT PART)

---

### 3.1 Disable Elasticsearch Output (IMPORTANT)

Filebeat can send directly to ES, **but we want Logstash in between**.

Edit:

```bash
vi /etc/filebeat/filebeat.yml
```

Comment out Elasticsearch output:

```yaml
#output.elasticsearch:
#  hosts: ["localhost:9200"]
```

---

### 3.2 Enable Logstash Output

Add / modify:

```yaml
output.logstash:
  hosts: ["logstash-node:5044"]
```

Explanation:
• Filebeat sends logs to Logstash
• 5044 is default Beats port

---

### 3.3 Enable System Module (EASY & REAL LOGS)

Filebeat modules give **structured logs**.

Enable system module:

```bash
filebeat modules enable system
```

This module automatically handles:
• syslog
• auth logs
• timestamps
• severity

---

### 3.4 Configure System Module Paths

Edit:

```bash
vi /etc/filebeat/modules.d/system.yml
```

Set:

```yaml
- module: system
  syslog:
    enabled: true
    var.paths:
      - /var/log/messages
  auth:
    enabled: true
    var.paths:
      - /var/log/secure
```

---

## 4. Configure Logstash to Receive Filebeat

Now we must **update Logstash pipeline**.

---

### 4.1 Create Beats Input Pipeline

Edit:

```bash
vi /etc/logstash/conf.d/filebeat.conf
```

Add:

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  # no filter for now (raw logs)
}

output {
  elasticsearch {
    hosts => ["http://es-node-1:9200","http://es-node-2:9200","http://es-node-3:9200"]
    index => "filebeat-%{+YYYY.MM.dd}"
  }
}
```

---

### 4.2 Test Logstash Config

```bash
/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Expected:

```text
Configuration OK
```

---

### 4.3 Restart Logstash

```bash
systemctl restart logstash
systemctl status logstash
```

---

## 5. Start Filebeat

---

### 5.1 Test Filebeat Configuration

```bash
filebeat test config
```

Expected:

```text
Config OK
```

---

### 5.2 Test Output Connection

```bash
filebeat test output
```

Expected:

```text
logstash: logstash-node:5044... OK
```

---

### 5.3 Enable & Start Filebeat

```bash
systemctl enable filebeat
systemctl start filebeat
```

Check status:

```bash
systemctl status filebeat
```

---

## 6. Verify Data Flow (VERY IMPORTANT)

---

### 6.1 Check Logstash Logs

```bash
journalctl -u logstash -f
```

You should see:
• Beats connection
• Events received

---

### 6.2 Check Elasticsearch Indices

From any ES node:

```bash
curl http://localhost:9200/_cat/indices?v
```

You should see:

```text
filebeat-YYYY.MM.dd
```

---

## 7. Verify in Kibana

---

### 7.1 Create Index Pattern

In Kibana UI:
• Stack Management → Index Patterns
• Create index pattern:

```text
filebeat-*
```

• Time field: `@timestamp`

---

### 7.2 View Logs

• Go to **Discover**
• Select `filebeat-*`
• You should see:

* SSH logins
* sudo usage
* system events

🎉 These are **REAL logs from your server**.

---

## 8. Generate Test Logs (For Confirmation)

Run on logstash-node:

```bash
logger "Filebeat test log from ELK lab"
```

Check Kibana → Discover
You should see the message.

---

## 9. Common Beginner Problems & Fixes

### No logs in Kibana

• Filebeat not running
• Wrong Logstash host
• Port 5044 blocked
• Logstash pipeline not loaded

---

### Filebeat running but no data

• Wrong log path
• SELinux blocking (set permissive for lab):

```bash
setenforce 0
```

---

## 10. What You Have Achieved

Now your ELK stack is **fully functional**:

✔ Real system logs
✔ Filebeat agent
✔ Logstash ingestion
✔ Elasticsearch indexing
✔ Kibana visualization

This is **production-grade ELK knowledge**.

---

## Next Steps (Highly Valuable)

Choose what you want next:

1. **Grok filters** for structured parsing
2. Filebeat on **remote servers**
3. Enable **Elastic Security (TLS, users)**
4. Index Lifecycle Management (ILM)
5. Dashboards for system logs

Tell me your next step and we’ll go deeper 🚀
