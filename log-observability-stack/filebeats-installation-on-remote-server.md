# Install Filebeat on Remote Servers (From Zero)


## 0. Architecture

```
[Remote Server 1]      [Remote Server 2]      [Remote Server N]
     Filebeat               Filebeat               Filebeat
         |                      |                      |
         +----------------------+----------------------+
                                |
                           (Beats / 5044)
                                |
                          [Logstash Node]
                                |
                        [Elasticsearch Cluster]
                                |
                             [Kibana]
```

Key idea:
- Remote servers **do NOT talk to Elasticsearch**
- Only Logstash is exposed
- Centralized control

---

## 1. Prepare Remote Server

Assume:
- CentOS / RHEL / Rocky / Alma
- IP example: `192.168.10.50`
- Hostname: `app-server-1`


### 1.1 Set Remote Server Hostname

```bash
hostnamectl set-hostname app-server-1
#
reboot
```


### 1.2 Update System

```bash
dnf update -y
```

---

### 1.3 Configure `/etc/hosts` for Logstash (VERY IMPORTANT)

```bash
vi /etc/hosts
```

Add:

```text
192.168.10.30  logstash-node
```

>You do **NOT** need ES or Kibana here.

---

## 2. Install Filebeat on Remote Server


### 2.1 Import Elastic GPG Key

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```


### 2.2 Add Elastic Repository

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


### 2.3 Install Filebeat

```bash
dnf install filebeat -y
```

---

## 3. Configure Filebeat on Remote Server

### 3.1 Disable Elasticsearch Output

**Edit:**

```bash
vi /etc/filebeat/filebeat.yml
```

**Comment out:**

```yaml
#output.elasticsearch:
#  hosts: ["localhost:9200"]
```


### 3.2 Enable Logstash Output

Add:

```yaml
output.logstash:
  hosts: ["logstash-node:5044"]
```



### 3.3 Enable System Module (Recommended)

```bash
filebeat modules enable system
```


### 3.4 Configure Log Paths

Edit:

```bash
vi /etc/filebeat/modules.d/system.yml
```

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

## 4. Firewall Configuration

### 4.1 Remote Server

Filebeat does **outbound only**, so:
- No firewall port needed on remote server


### 4.2 Logstash Node (IMPORTANT)

Ensure port 5044 is open:

```bash
firewall-cmd --permanent --add-port=5044/tcp
firewall-cmd --reload
```

---

## 5. Configure Logstash

You already have this, but verify:

```bash
vi /etc/logstash/conf.d/filebeat.conf
```

```conf
input {
  beats {
    port => 5044
  }
}

filter {
}

output {
  elasticsearch {
    hosts => ["http://es-node-1:9200","http://es-node-2:9200","http://es-node-3:9200"]
    index => "filebeat-%{+YYYY.MM.dd}"
  }
}
```

Test:

```bash
/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Restart:

```bash
systemctl restart logstash
```

---

## 6. Start Filebeat on Remote Server

### 6.1 Test Configuration

```bash
filebeat test config
```

Expected:

```text
Config OK
```


### 6.2 Test Output

```bash
filebeat test output
```

Expected:

```text
logstash: logstash-node:5044... OK
```

---

### 6.3 Enable & Start

```bash
systemctl enable filebeat
systemctl start filebeat
systemctl status filebeat
```

---

## 7. Verify Logs from Remote Server


### 7.1 Generate Test Log

```bash
logger "Hello from remote server app-server-1"
```


### 7.2 Verify in Kibana

In **Kibana → Discover**:
- Index pattern: `filebeat-*`
- Filter:

```text
host.name : "app-server-1"
```

You should see the log.

---

## 8. Identify Remote Servers in Kibana

Filebeat automatically adds:
- `host.name`
- `host.ip`
- `agent.name`

This allows:
- Filtering per server
- Dashboards per environment

---

## 9. Common Production Mistakes

### Logs not arriving

• Port 5044 blocked
• Wrong hostname
• SELinux blocking

Temporarily (lab only):

```bash
setenforce 0
```


### Filebeat stopped sending logs

• Logstash down
• Network issue
• File permissions

Check:

```bash
journalctl -u filebeat -f
```

---

## 10. Scaling to Many Servers (IMPORTANT)

Real production:
• Hundreds of Filebeat agents
• One or more Logstash nodes
• Load balancer in front of Logstash

Example:

```yaml
output.logstash:
  hosts: ["ls-1:5044","ls-2:5044"]
  loadbalance: true
```
---
