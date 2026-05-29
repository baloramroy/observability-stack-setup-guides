# Kibana Secure Configuration & Elasticsearch Integration

This SOP focuses only on **TLS trust + secure communication**, not installation.

```text
Browser (HTTPS) → Kibana (HTTPS) → Elasticsearch (HTTPS)
```

#

## Scope of This SOP

This SOP covers secure **Kibana integration** with an existing secured Elasticsearch cluster.

This document includes:

* Elasticsearch **CA trust** configuration
* Kibana **HTTPS browser certificate** configuration
* **Kibana secure connection** configuration
* **Kibana authentication** configuration
* **HTTPS** communication validation
* **Firewall** configuration
* Kibana **startup and verification**

This SOP assumes:

* Elasticsearch cluster is already operational
* Elasticsearch HTTP TLS is already configured
* Elasticsearch certificates are already generated
* Elasticsearch security is enabled

This SOP does NOT cover:

* Kibana installation
* Elasticsearch installation
* Elasticsearch TLS generation

---

## Lab Environment

| Hostname    | IP            | Role          |
| ----------- | ------------- | ------------- |
| es-node-1   | 192.168.0.124 | Elasticsearch |
| es-node-2   | 192.168.0.125 | Elasticsearch |
| es-node-3   | 192.168.0.126 | Elasticsearch |
| kibana-node | 192.168.0.123 | Kibana        |

---

## Target Architecture

```text
[ Browser ]
     │ HTTPS
     ▼
[ Kibana Server ]
     │ HTTPS
     ▼
[ Elasticsearch Cluster ]
```


---

## TLS Roles (Very Important Concept)

| Component     | Role            | Needs Certificate | Needs CA Trust          |
| ------------- | --------------- | ----------------- | ----------------------- |
| Browser       | Client          | No                | Yes (Kibana CA)         |
| Kibana        | Server + Client | Yes (server cert) | Yes (Elasticsearch CA)  |
| Elasticsearch | Server          | Yes (server cert) | No (optional mTLS only) |

---

## Elasticsearch Security Architecture

The Elasticsearch cluster already uses:

* **HTTPS/TLS** on port `9200`
* **Transport TLS** on port `9300`
* Internal security enabled
* Certificate-based trust

Kibana must therefore:

* Communicate using **HTTPS**
* Trust the **Elasticsearch CA**
* Authenticate using **Elasticsearch credentials**

---

## Certificate Strategy (Production Grade)

We will maintain **two trust domains**:

### 1. Elasticsearch PKI Domain

* CA: `elastic-stack-ca`
* Used for:

  * Elasticsearch HTTP TLS (9200)
  * Elasticsearch Transport TLS (9300)
  * Kibana trust validation

### 2. Kibana PKI Domain

* CA: `kibana-ca`
* Used for:

  * Kibana HTTPS (5601)
  * Browser trust

---

## Important Kibana Paths

| Path                     | Purpose                             |
| ------------------------ | ----------------------------------- |
| `/etc/kibana/`           | Main Kibana configuration directory |
| `/etc/kibana/kibana.yml` | Main Kibana configuration           |
| `/etc/kibana/certs/`     | Certificate directory               |
| `/var/log/kibana/`       | Kibana logs                         |

---


## Prepare Elasticsearch CA Certificate

Kibana must **trust** the **CA** that **signed** Elasticsearch **HTTP** certificates.


### Create Certificate Directory

Run on `kibana-node`:

```bash
mkdir -p /etc/kibana/certs/{elastic-certs,kibana-certs}
```

#

### Copy Elasticsearch CA Certificate

From `es-node-1`:

```bash
scp /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
root@192.168.0.123:/etc/kibana/certs/elastic-certs/
```




#

### Set Ownership & Permissions

Run on `kibana-node`:

- Set Ownership

  ```bash
  chown -R root:kibana /etc/kibana/certs
  ```

- Set Permissions

  ```bash
  find /etc/kibana/certs -type d -exec chmod 750 {} \;
  find /etc/kibana/certs -type f -exec chmod 640 {} \;
  ```

---

# 2. Create Kibana System User Password

Kibana should use the built-in `kibana_system` user for Elasticsearch communication.

- Run on ONE Elasticsearch node:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-reset-password \
  -u kibana_system
  ```

- Example output:

  ```text
  New value: xxxxxxxxxxxxx
  ```

  Save this password securely.

---

## Configure Kibana



### 3.1 Edit Kibana Configuration

Run on `kibana-node`:

```bash
vi /etc/kibana/kibana.yml
```

---

### 3.2 Configure Kibana Secure Settings

Add or modify:

```yaml
# ---------------------------------------
# Kibana Server
# ---------------------------------------

server.port: 5601
server.host: "0.0.0.0"

server.name: "kibana-node"

# ---------------------------------------
# Elasticsearch Connection
# ---------------------------------------

elasticsearch.hosts:
  - "https://es-node-1:9200"
  - "https://es-node-2:9200"
  - "https://es-node-3:9200"

# ---------------------------------------
# Kibana Authentication
# ---------------------------------------

elasticsearch.username: "kibana_system"
elasticsearch.password: "KIBANA_SYSTEM_PASSWORD"

# ---------------------------------------
# Elasticsearch TLS Trust
# ---------------------------------------

elasticsearch.ssl.truststore.path: /etc/kibana/certs/elastic-stack-ca.p12
```


### Configuration Explanation

| Setting                | Purpose                            |
| ---------------------- | ---------------------------------- |
| `server.host: 0.0.0.0` | Allows remote browser access       |
| `elasticsearch.hosts`  | Elasticsearch HTTPS endpoints      |
| `kibana_system`        | Internal Kibana service account    |
| `truststore.path`      | Trust Elasticsearch CA certificate |

---

## Verify Elasticsearch HTTPS Connectivity

Before starting Kibana, verify HTTPS communication from the Kibana server.

Run on `kibana-node`:

```bash
curl -k -u elastic https://es-node-1:9200
```

Expected output:

```json
{
  "name" : "es-node-1",
  "cluster_name" : "elk-prod-cluster",
  ...
}
```

---

## Better Production Verification

Instead of `-k`, use trusted CA verification:

```bash
curl --cacert /etc/kibana/certs/http_ca.crt \
-u elastic https://es-node-1:9200
```


---

## Kibana browser trust configuration


### Create Kibana CA

- Run on a secure admin machine:

  ```bash
  mkdir -p /root/kibana-ca
  cd /root/kibana-ca
  ```

- Generate CA:

  ```bash
  openssl genrsa -out kibana-ca.key 4096
  openssl req -x509 -new -nodes -key kibana-ca.key -sha256 -days 3650 -out kibana-ca.crt
  ```

  You now have:

  ```text
  kibana-ca.crt   (public)
  kibana-ca.key   (private - KEEP SAFE)
  ```

---

### Create Kibana Server Certificate

- On CA machine:

  ```bash
  openssl genrsa -out kibana-server.key 2048
  ```

- Create CSR:

  ```bash
  openssl req -new -key kibana-server.key -out kibana-server.csr
  ```

- Sign it:

  ```bash
  openssl x509 -req -in kibana-server.csr \
    -CA kibana-ca.crt -CAkey kibana-ca.key -CAcreateserial \
    -out kibana-server.crt -days 825 -sha256
  ```

  Now you have:

  ```text
  kibana-server.crt
  kibana-server.key
  ```

---

### Install Kibana Certificates

- On Kibana server:

  ```bash
  mkdir -p /etc/kibana/certs
  ```

- Copy:

  ```bash
  kibana-server.crt
  kibana-server.key
  kibana-ca.crt
  ```


- Set Permissions

  ```bash
  chown -R root:kibana /etc/kibana/certs
  find /etc/kibana/certs -type d -exec chmod 750 {} \;
  find /etc/kibana/certs -type f -exec chmod 640 {} \;
  ```

---

### Enable Kibana HTTPS (Browser Security)

- Edit:

  ```bash
  /etc/kibana/kibana.yml
  ```

- Add:

  ```yaml
  server.ssl.enabled: true
  server.ssl.certificate: /etc/kibana/certs/kibana-server.crt
  server.ssl.key: /etc/kibana/certs/kibana-server.key
  ```




---
## Firewall Rules

On Kibana Node:

```bash
firewall-cmd --permanent --add-port=5601/tcp
firewall-cmd --reload
```

On Elasticsearch Node:

```bash
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --reload
```

---

## Start Kibana Service

Reload Systemd

```bash
systemctl daemon-reload
```

Enable Kibana Service

```bash
systemctl enable kibana
```

Start Kibana

```bash
systemctl start kibana
```

Verify Kibana Service status:

```bash
systemctl status kibana
```

Expected:

```text
active (running)
```

---

## Monitor Kibana Logs

If Kibana fails:

```bash
journalctl -u kibana -f
```

---

## 14. Final Secure Flow Verification

### Step 1 — Browser → Kibana

```text
https://kibana-node:5601
```

Check:

* No certificate warning
* Valid HTTPS lock

#

### Step 2 — Kibana → Elasticsearch

Check logs:

```bash
journalctl -u kibana -f
```

Expected:

```text
Connected to Elasticsearch cluster
TLS handshake successful
```

---

## Verify Kibana ↔ Elasticsearch Integration

Inside Kibana:

* Open:

  * Stack Management
  * Monitoring
  * Dev Tools

Verify:

* cluster status
* node visibility
* index visibility

---

## Common Beginner Mistakes

* Using HTTP instead of HTTPS
* Wrong certificate permissions
* Wrong hostname/IP in certificates
* Incorrect kibana_system password
* Firewall blocking port 5601
* Elasticsearch cluster unhealthy
* Wrong CA path

---

## Important Production Recommendations

Production deployments should ideally use:

* service account tokens
* dedicated Kibana certificates
* reverse proxy/load balancer
* DNS instead of `/etc/hosts`

---

## Summary

This SOP covered:

* Kibana secure configuration
* Elasticsearch CA trust
* Kibana authentication
* HTTPS communication
* Kibana startup and verification

---
