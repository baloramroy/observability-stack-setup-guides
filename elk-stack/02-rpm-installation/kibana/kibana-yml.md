# Kibana Secure Configuration & Elasticsearch Integration

## Purpose

This SOP describes how to securely **integrate Kibana** with an **existing secured Elasticsearch cluster** using **TLS encryption** and **authentication**.

This document covers:

* Elasticsearch CA trust configuration
* Kibana HTTPS configuration
* Kibana authentication configuration
* Kibana keystore configuration
* Kibana encryption keys
* DNS and certificate validation
* Firewall configuration
* Kibana startup and verification
* Kibana ↔ Elasticsearch secure communication validation

This SOP assumes:

* Kibana package is already installed
* Elasticsearch cluster is already operational
* Elasticsearch security is enabled
* Elasticsearch HTTP TLS is already configured
* Elasticsearch transport TLS is already configured
* Elasticsearch certificates are already generated

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

Understanding certificate trust is critical.

### Browser → Kibana

Kibana acts as a server.

Requirements:

* Kibana presents a server certificate.
* Browser validates the Kibana certificate.
* Browser trusts the CA that signed the Kibana certificate.

#

### Kibana → Elasticsearch

Kibana acts as a client.

Requirements:

* Elasticsearch presents a server certificate.
* Kibana validates the Elasticsearch certificate.
* Kibana trusts the Elasticsearch CA.

#

### TLS Role Summary

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

---

## Elasticsearch Prerequisites Validation

Before proceeding, verify:

## Cluster Health

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

```bash
curl -k -u elastic:pass https://es-node-1:9200
```

- Expected:

  ```json
  {
    "name":"es-node-1"
  }
  ```

Successful authentication confirms:

* HTTPS is working
* Security is enabled
* Elasticsearch is accessible

---

## Required Certificate Artifacts

This SOP assumes the following files are already available from Appendix A or Appendix B.

  - kibana-server.crt
  - kibana-server.key
  - elasticsearch-ca.pem

---

## Create Certificate Directories

- Create certificate directories.

  ```bash
  mkdir -p /etc/kibana/certs/{elastic-certs,kibana-certs}
  ```

- Expected layout:

  ```text
  /etc/kibana/certs/

  ├── elastic-certs/
  │   └── elasticsearch-ca.pem
  │
  └── kibana-certs/
      ├── kibana-server.crt
      └── kibana-server.key
  ```

---

## Deploy Elasticsearch CA Trust

Kibana must trust the **CA** that **signed Elasticsearch HTTPs** certificates.

- Copy the CA certificate to Kibana.

  ```bash
  scp /etc/elasticsearch/certs/zipcert/kibana/elasticsearch-ca.pem \
  root@192.168.0.123:/etc/kibana/certs/elastic-certs/
  ```

- Only the **public CA certificate** should be copied.

- Never copy:

  * CA private keys
  * Elasticsearch private keys
  * PKCS12 bundles containing private keys

>[!Note]
If `elasticsearch-ca.pem` is unavailable, refer to the **Elasticsearch Certificate Management SOP**.

---

## Deploy Kibana HTTPS Certificate

- Copy files from `es-node-1`:

  ```bash
  scp /root/kibana-certutil/kibana-cert/kibana-server.crt root@192.168.0.123:/etc/kibana/certs/kibana-certs/

  scp /root/kibana-certutil/kibana-cert/kibana-server.key root@192.168.0.123:/etc/kibana/certs/kibana-certs/
  ```
  
---

## Set Ownership

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
  namei -l /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem
  ```

  >Verify the kibana group can access the certificate.

---

## Browser Trust Requirements

For browsers to trust **Kibana HTTPS**, they must **trust the CA** that signed the **Kibana server certificate**.


## Windows

- Import:

  ```text
  kibana-ca.crt
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

---

## RHEL / Rocky / AlmaLinux

- Copy Certificate in this path:

  ```bash
  cp kibana-ca.crt \
  /etc/pki/ca-trust/source/anchors/
  ```

- Then Run:

  ```bash
  update-ca-trust
  ```

---

## Ubuntu
- Copy Certificate in this path:

  ```bash
  cp kibana-ca.crt \
  /usr/local/share/ca-certificates/
  ```

- Then Run:

  ```bash
  update-ca-certificates
  ```

---

## Create Kibana Authentication Credentials

Kibana requires authentication to communicate with Elasticsearch.


### Reset kibana_system Password

- Run on any Elasticsearch node.

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-reset-password \
  -u kibana_system
  ```

- Example:

  ```text
  New value: XXXXXXXXXXXXX
  ```

Store this password securely.

---

## Create Kibana Keystore

Sensitive values should never be stored directly inside `kibana.yml`.


### Create Keystore

```bash
/usr/share/kibana/bin/kibana-keystore create
```


### Add Kibana Password

```bash
/usr/share/kibana/bin/kibana-keystore add elasticsearch.password
```

Enter:

```text
kibana_system password
```


### Verify Keystore Exists

```bash
ls -l /etc/kibana/
```

Expected:

```text
kibana.keystore
```

---


## Generate Kibana Encryption Keys

Kibana requires encryption keys for secure operation.

These keys must remain constant across restarts.

#

## Generate Keys

```bash
/usr/share/kibana/bin/kibana-encryption-keys generate
```

Example output:

```yaml
xpack.security.encryptionKey:
xpack.encryptedSavedObjects.encryptionKey:
xpack.reporting.encryptionKey:
```

Save the generated values securely.

---

## Configure Kibana.yml

Edit:

```bash
vi /etc/kibana/kibana.yml
```



### Kibana Server Configuration

```yaml
server.port: 5601

server.host: "0.0.0.0"

server.name: "kibana-node"
```


### Enable HTTPS

```yaml
server.ssl.enabled: true

server.ssl.certificate: /etc/kibana/certs/kibana-certs/kibana-server.crt

server.ssl.key: /etc/kibana/certs/kibana-certs/kibana-server.key
```


### Elasticsearch Connection

```yaml
elasticsearch.hosts:
  - "https://es-node-1:9200"
  - "https://es-node-2:9200"
  - "https://es-node-3:9200"
```


### Authentication

```yaml
elasticsearch.username: "kibana_system"
```

Password is stored in:

```text
kibana keystore
```


### Elasticsearch CA Trust

```yaml
elasticsearch.ssl.certificateAuthorities:
  - /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem
```


### Certificate Verification

```yaml
elasticsearch.ssl.verificationMode: full
```

`full` verifies:

* Certificate validity
* CA trust
* Hostname validation


### Kibana Encryption Keys

```yaml
xpack.security.encryptionKey: "<generated-key>"

xpack.encryptedSavedObjects.encryptionKey: "<generated-key>"

xpack.reporting.encryptionKey: "<generated-key>"
```

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
  --cacert /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem \
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
  -CAfile /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem
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


## Production Recommendations

For production deployments:

* Use DNS instead of `/etc/hosts`
* Use certificates containing proper SAN entries
* Store secrets in Kibana keystore
* Regularly rotate certificates
* Regularly rotate service credentials
* Use NTP time synchronization
* Restrict certificate file permissions
* Use dedicated monitoring
* Use load balancers for high availability
* Use service account tokens instead of passwords when organizational standards permit

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

