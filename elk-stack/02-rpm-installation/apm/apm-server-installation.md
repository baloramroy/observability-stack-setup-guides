# SOP: Installation and Initial Configuration of Elastic APM Server on Linux

---

## Purpose

This SOP describes the procedure to:

1. Install Elastic APM Server using RPM/YUM/DNF package manager
2. Configure connectivity with Elasticsearch
3. Start and validate APM Server service
4. Verify APM data ingestion readiness

---

## Scope

Applicable for:

* RHEL 7 / 8 / 9
* CentOS 7 / Stream 8 / Stream 9
* Rocky Linux
* AlmaLinux

---

## Prerequisites

Before starting, ensure:

| Requirement          | Description             |
| -------------------- | ----------------------- |
| Elasticsearch        | Running and reachable   |
| Kibana               | Running and reachable   |
| Network Connectivity | Required ports are open |
| Root/Sudo Access     | Required                |

---

## Required Network Connectivity

| Source             | Destination   | Port                    | Purpose               |
| ------------------ | ------------- | ----------------------- | --------------------- |
| APM Server         | Elasticsearch | 10217 / Custom HTTP Port | Data ingestion       |


Example:

```text
Application → APM Server :8200
APM Server → Elasticsearch :10217
```

---

## Configure Elastic Repo

Create a repo file:
```bash
vi /etc/yum.repos.d/elasticsearch.repo
```

Add these lines:

```ini
[elasticsearch]
name=Elasticsearch repository
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
type=rpm-md
```

---

## Verify Elastic Repository Configuration

Check repository:

```bash
dnf repolist | grep elastic
```

OR

```bash
yum repolist | grep elastic
```

Expected:

```text
elastic-6.x
```

---

## Install APM Server Package

To Check all Available **APM-Server** Version:

```bash
dnf list --showduplicates apm-server
```

Now install **apm-server** required version from the repository:

```bash
dnf clean all
dnf makecache --refresh
dnf install apm-server-6.5.2-1
```

⚠ Do NOT start it yet.


---

# 8. Important File Locations

| Path                             | Purpose            |
| -------------------------------- | ------------------ |
| `/etc/apm-server/apm-server.yml` | Main configuration |
| `/var/log/apm-server/`           | Logs               |
| `/usr/share/apm-server/`         | Binary files       |
| `/var/lib/apm-server/`           | Data directory     |
| `/etc/systemd/system/`           | Service files      |

---

## Backup Default Configuration

Before modification:

```bash
cp -p /etc/apm-server/apm-server.yml /etc/apm-server/apm-server.yml.bak
```

---

## Configure APM Server

Edit configuration:

```bash
vim /etc/apm-server/apm-server.yml
```

Add below line:

```yml
apm-server:
  host: "localhost:8200"

setup.template.settings:
  index:
    number_of_shards: 1
    codec: best_compression

output.elasticsearch:
  hosts: ["nagad-dmz-mon1:10217"]
  indices:
    - index: "apm-%{[beat.version]}-sourcemap"
      when.contains:
        processor.event: "sourcemap"
    - index: "apm-%{[beat.version]}-error-%{+yyyy.MM.dd}"
      when.contains:
        processor.event: "error"
    - index: "apm-%{[beat.version]}-transaction-%{+yyyy.MM.dd}"
      when.contains:
        processor.event: "transaction"
    - index: "apm-%{[beat.version]}-span-%{+yyyy.MM.dd}"
      when.contains:
        processor.event: "span"
    - index: "apm-%{[beat.version]}-metric-%{+yyyy.MM.dd}"
      when.contains:
        processor.event: "metric"
    - index: "apm-%{[beat.version]}-onboarding-%{+yyyy.MM.dd}"
      when.contains:
        processor.event: "onboarding"

logging.metrics.enabled: false

```
---

## Validate Configuration Syntax

Run:

```bash
apm-server test config
```

Expected:

```text
Config OK
```

---

## Test Elasticsearch Connectivity

Run:

```bash
apm-server test output
```

Expected:

```text
elasticsearch: https://10.110.10.21:10217...
  parse url... OK
  connection... OK
  version... 8.x.x
```

---

## Enable and Start APM Server Service

Reload Systemd

```bash
systemctl daemon-reload
```

Enable Service

```bash
systemctl enable apm-server
```

Start Service

```bash
systemctl start apm-server
```

---

## Verify Service Status

```bash
systemctl status apm-server
```

Expected:

```text
active (running)
```

---

## Verify Listening Port

Run:

```bash
ss -tulnp | grep 8200
```

Expected:

```text
LISTEN 0 128 0.0.0.0:8200
```

---

# 20. Verify APM Server Logs

View logs:

```bash
journalctl -u apm-server -f
```

OR

```bash
tail -f /var/log/apm-server/apm-server
```

---

## Configure Firewall

Open Port 8200

```bash
firewall-cmd --permanent --add-port=8200/tcp
```

Reload Firewall

```bash
firewall-cmd --reload
```

Verify

```bash
firewall-cmd --list-ports
```

---


## Validate APM Server Endpoint

From application server:

```bash
curl http://localhost:8200/
```

Expected:

```json
{
  "build_date":"...",
  "build_sha":"...",
  "publish_ready":true,
  "version":"8.x.x"
}
```


---

## Typical Minimal Working Configuration

Example:

```yaml
apm-server:
  host: "0.0.0.0:8200"

output.elasticsearch:
  hosts: ["https://10.110.10.21:10217"]

  username: "elastic"
  password: "PASSWORD"

  ssl.enabled: true
  ssl.certificate_authorities:
    - /etc/apm-server/certs/elasticsearch-ca.crt

setup.kibana:
  host: "https://10.110.10.22:5601"
```

---

## Post-Installation Validation Checklist

| Validation                    | Status |
| ----------------------------- | ------ |
| Package Installed             | ☐      |
| Service Running               | ☐      |
| Port 8200 Listening           | ☐      |
| Elasticsearch Connectivity OK | ☐      |
| Kibana Connectivity OK        | ☐      |
| TLS Working                   | ☐      |
| No Error in Logs              | ☐      |
| Agent Connectivity Working    | ☐      |
| APM Data Visible in Kibana    | ☐      |

---

## Expected Kibana Verification

Navigate in Kibana:

```text
Observability → APM → Services
```

Expected:

* Services visible
* Transactions visible
* JVM metrics visible
* Trace data visible

---

## Rollback Procedure

## Stop Service

```bash
systemctl stop apm-server
```

## Disable Service

```bash
systemctl disable apm-server
```

## Remove Package

```bash
dnf remove apm-server
```

OR

```bash
yum remove apm-server
```

---

