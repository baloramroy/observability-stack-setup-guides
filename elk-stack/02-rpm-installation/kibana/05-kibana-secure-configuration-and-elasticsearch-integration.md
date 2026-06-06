# Kibana Secure Configuration & Elasticsearch Integration

## Purpose

This SOP describes how to securely **integrate Kibana** with an **existing secured Elasticsearch cluster** using **TLS encryption** and **authentication**.

This document covers:

* Elasticsearch **CA trust** configuration
* Kibana **HTTPS** configuration
* Kibana **authentication** configuration
* Kibana **keystore** configuration
* Kibana **encryption** keys
* **DNS and certificate** validation
* **Firewall** configuration
* Kibana **startup and verification**
* Kibana ↔ Elasticsearch **secure communication** validation

This SOP does NOT cover:

* Kibana installation
* Elasticsearch installation
* Elasticsearch cluster creation
* Elasticsearch TLS certificate generation


---

## Lab Environment

| Hostname    | IP Address    | Role          |
| ----------- | ------------- | ------------- |
| es-node-1   | 192.168.0.124 | Elasticsearch |
| es-node-2   | 192.168.0.125 | Elasticsearch |
| es-node-3   | 192.168.0.126 | Elasticsearch |
| kibana-node | 192.168.0.123 | Kibana        |

---

## Architecture

```text
              HTTPS
+------------------------------+
|           Browser            |
+------------------------------+
               │
               ▼
+------------------------------+
|         Kibana Server        |
|          Port 5601           |
+------------------------------+
               │
               │ HTTPS
               ▼
+------------------------------+
|   Elasticsearch Cluster      |
|          Port 9200           |
+------------------------------+
```

---

## TLS Trust Model

**Understanding certificate trust is critical.**

### ⬛ Browser → Kibana

Kibana acts as a **server**.

Requirements:

* **Kibana** presents a **server** certificate.
* **Browser** validates the **Kibana certificate**.
* **Browser** trusts the **CA** that **signed the Kibana certificate**.

#

### ⬛ Kibana → Elasticsearch

Kibana acts as a **client**.

Requirements:

* **Elasticsearch** presents a **server** certificate.
* **Kibana** validates the **Elasticsearch certificate**.
* Kibana **trusts** the **Elasticsearch CA**.

#

### ⬛ TLS Role Summary

| Component     | Role            | Server Certificate | CA Trust Required |
| ------------- | --------------- | ------------------ | ----------------- |
| Browser       | Client          | No                 | Kibana CA         |
| Kibana        | Server + Client | Yes                | Elasticsearch CA  |
| Elasticsearch | Server          | Yes                | No                |

---

## Prerequisites

Before starting this SOP, ensure:

- **Kibana** package is **installed**
- Elasticsearch **cluster** is **operational**
- Elasticsearch **security** is enabled
- Elasticsearch **HTTP TLS** is configured
- Elasticsearch **transport TLS** is configured
- Kibana **HTTPS certificate** has already been generated
- Kibana **private key** has already been generated
- Elasticsearch **CA certificate** is available
- **Authentication** method ready (SOP-4)
  
---

## Elasticsearch Prerequisites Validation

Before proceeding, verify:

## Cluster Health

- Run:

  ```bash
  curl -k -u elastic:pass https://es-node-1:9200/_cluster/health?pretty
  ```

- Expected:

  ```json
  {
  "status" : "green"
  }
  ```

#

## Security Status

- Run:

  ```bash
  curl -k -u elastic:pass https://es-node-1:9200
  ```

- Expected:

  ```json
  {
    "name":"es-node-1"
  }
  ```

- Successful authentication confirms:

  * HTTPS is working
  * Security is enabled
  * Elasticsearch is accessible

---

## Required Certificate Artifacts

This SOP assumes the following files are already available from Kibana TLS generation SOP.

  - kibana-server.crt
  - kibana-server.key
  - elasticsearch-ca.pem

---

## Create Certificate Directories

- Create certificate directories.

  ```bash
  mkdir -p /etc/kibana/certs/{elastic-ca,kibana-certs}
  ```

- Expected layout:

  ```text
  /etc/kibana/certs/

  ├── elastic-ca/
  │   └── elasticsearch-ca.pem
  │
  └── kibana-certs/
      ├── kibana-server.crt
      └── kibana-server.key
  ```

---

## Deploy Elasticsearch CA Certificate

Kibana must trust the **CA** that **signed Elasticsearch HTTPs** certificates.

- Copy the CA certificate to Kibana.

  ```bash
  scp /etc/elasticsearch/certs/zipcert/kibana/elasticsearch-ca.pem \
  root@192.168.0.123:/etc/kibana/certs/elastic-ca/
  ```

- Only the **public CA certificate** should be copied.


>[!Note]
If `elasticsearch-ca.pem` is unavailable, refer to the -> [03-elasticsearch-tls-certificate-generation.md](../elasticsearch/03-elasticsearch-tls-certificate-generation.md) SOP.

---

## Deploy Kibana HTTPS Certificate

- Copy Certificate from `es-node-1`:

  ```bash
  scp /root/kibana-certutil/kibana-cert/kibana-server.crt root@192.168.0.123:/etc/kibana/certs/kibana-certs/
  ```

- Copy Certificate Keys from `es-node-1`:

  ```bash
  scp /root/kibana-certutil/kibana-cert/kibana-server.key root@192.168.0.123:/etc/kibana/certs/kibana-certs/
  ```
  
---

## Set Ownership of Copied Certificate

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

- Validate Permissions

  ```bash
  namei -l /etc/kibana/certs/elastic-ca/elasticsearch-ca.pem
  namei -l /etc/kibana/certs/kibana-certs/kibana-server.crt
  namei -l /etc/kibana/certs/kibana-certs/kibana-server.key
  ```

  >Verify the kibana group can access the certificate.

---

## Browser Trust Requirements

For browsers to trust **Kibana HTTPS**, they must **trust the CA** that signed the **Kibana server certificate**.

#

### Windows

- Import the CA certificate that signed the Kibana server certificate.:

  ```text
  elasticsearch-ca.pem
  ```

- Into:

  ```text
  Trusted Root Certification Authorities
  ```

- Process using MMC:

  This method provides a visual interface to manage certificates.

  - Press `Windows + R`, type `mmc`, and press **Enter**.
  - In the console, go to `File > Add/Remove Snap-in`.
  - Select `Certificates` from the list and click `Add >`.
  - Choose **Computer** account, then **Next**. Select **Local Computer**, click **Finish**, and then **OK**.
  - In the left panel, expand `Certificates (Local Computer)` > `Trusted Root Certification Authorities`.
  - Right-click on **Certificates**, go to **All Tasks**, and select **Import....**
  - Click **Next**, browse to your **certificate** file, and follow the **prompts**. Ensure the wizard places it in the `Trusted Root Certification Authorities store`.

#

### RHEL / Rocky / AlmaLinux

- Copy Certificate in this path:

  ```bash
  cp /etc/kibana/certs/elastic-ca/elasticsearch-ca.pem \
  /etc/pki/ca-trust/source/anchors/
  ```

- Then Run:

  ```bash
  update-ca-trust
  ```

#

### Ubuntu
- Copy Certificate in this path:

  ```bash
  cp /etc/kibana/certs/elastic-ca/elasticsearch-ca.pem \
  /usr/local/share/ca-certificates/
  ```

- Then Run:

  ```bash
  update-ca-certificates
  ```

---


## Generate Kibana Encryption Keys

- Kibana requires encryption keys for secure operation.
- These keys must remain constant across restarts.


### ⬛ Generate Keys

- Run:

  ```bash
  /usr/share/kibana/bin/kibana-encryption-keys generate
  ```

- Example output:

  ```yaml
  xpack.security.encryptionKey: keys
  xpack.encryptedSavedObjects.encryptionKey: keys
  xpack.reporting.encryptionKey: keys
  ```

- Save the generated values securely.
- If this changes, all sessions become invalid

#

### ⬛ Real-World Example
  
- Without encryptionKey:

  - User logs in via HTTPS (TLS works ✅)
  - Kibana stores session in cookie (unencrypted cookie ❌)
  - Attacker steals cookie via XSS/browser cache
  - Attacker can impersonate user without credentials

- With encryptionKey:

  - Session data encrypted before cookie creation
  - Cookie theft yields unusable encrypted blob
  - Key only exists on Kibana server

---

## Configure `Kibana.yml` File

Edit `kibana.yml`:

```bash
vim /etc/kibana/kibana.yml
```

Insert all of this:

```yaml

### Kibana Server Configuration
server.port: 5601
server.host: "0.0.0.0"
server.name: "kibana-node"


### Enable HTTPS
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana-certs/kibana-server.crt
server.ssl.key: /etc/kibana/certs/kibana-certs/kibana-server.key


### Elasticsearch Connection
elasticsearch.hosts:
  - "https://es-node-1:9200"
  - "https://es-node-2:9200"
  - "https://es-node-3:9200"


# Authentication is defined in SOP-4
# Configure only ONE authentication method.
# Do not enable both simultaneously.
# elasticsearch.serviceAccountToken: ""
# OR
# elasticsearch.username/password (via keystore)

### Elasticsearch CA Trust
elasticsearch.ssl.certificateAuthorities:
  - /etc/kibana/certs/elastic-ca/elasticsearch-ca.pem


### Certificate Verification
elasticsearch.ssl.verificationMode: full


### Kibana Encryption Keys
xpack.security.encryptionKey: "<generated-key>"
xpack.encryptedSavedObjects.encryptionKey: "<generated-key>"
xpack.reporting.encryptionKey: "<generated-key>"

```
---

## Validate Configuration Syntax

- Run:

  ```bash
  /usr/share/kibana/bin/kibana --config /etc/kibana/kibana.yml --verbose
  or
  /usr/share/kibana/bin/kibana
  ```

  > Watch for configuration errors.

- This catches:

  - YAML mistakes
  - wrong file paths
  - invalid settings

  > before systemd startup.

---

## DNS Resolution Validation

Kibana must resolve Elasticsearch hostnames.

- Verify:

  ```bash
  getent hosts es-node-1
  getent hosts es-node-2
  getent hosts es-node-3
  ```

- Expected:

  ```text
  192.168.0.124 es-node-1
  192.168.0.125 es-node-2
  192.168.0.126 es-node-3
  ```

---

## TLS Validation Before Startup

Validate Elasticsearch certificate trust.

#

### Verify Elasticsearch HTTPS

- Run on Kibana:

  ```bash
  curl \
  --cacert /etc/kibana/certs/elastic-ca/elasticsearch-ca.pem \
  -u elastic \
  https://es-node-1:9200
  ```

- Expected:
  
  ```text
  Successful response without certificate errors.
  ```

#

### Verify TLS Handshake

- Run

  ```bash
  openssl s_client \
  -connect es-node-1:9200 \
  -servername es-node-1 \
  -CAfile /etc/kibana/certs/elastic-ca/elasticsearch-ca.pem
  ```

- Expected:

  ```text
  Verify return code: 0 (ok)
  ```


---

## Firewall Configuration

- Kibana Node

  ```bash
  firewall-cmd --permanent --add-port=5601/tcp
  ```

  ```bash
  firewall-cmd --reload
  ```

- Elasticsearch Nodes

  ```bash
  firewall-cmd --permanent --add-port=9200/tcp
  ```

  ```bash
  firewall-cmd --reload
  ```

---

## Pre-Startup Validation Checklist

- **Verify the following:**

  - ✓ kibana-server.crt exists
  - ✓ kibana-server.key exists
  - ✓ elasticsearch-ca.pem exists
  - ✓ Kibana can resolve Elasticsearch hostnames
  - ✓ Authentication method has been configured
  - ✓ Kibana encryption keys configured
  - ✓ kibana.yml syntax reviewed

- **After verify this start Kibana**.
---

## Start Kibana Service

- Reload systemd:

  ```bash
  systemctl daemon-reload
  ```

- Enable service:

  ```bash
  systemctl enable kibana
  ```

- Start service:

  ```bash
  systemctl start kibana
  ```

---

## Verify Service Status

```bash
systemctl status kibana
```

Expected:

```text
active (running)
```

---

## Summary

This SOP configured:

* Kibana HTTPS
* Elasticsearch CA trust
* Kibana authentication
* Kibana keystore
* Encryption keys
* TLS validation
* DNS validation
* Firewall configuration

The next documents in this series are:

---