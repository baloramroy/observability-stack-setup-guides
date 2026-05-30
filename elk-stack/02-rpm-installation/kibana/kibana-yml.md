# Kibana Secure Configuration & Elasticsearch Integration

## 1. Purpose of This SOP

This SOP describes how to securely integrate Kibana with an existing secured Elasticsearch cluster using TLS encryption and authentication.

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

Certificate generation for Kibana HTTPS will be covered separately in:

* Appendix A – Generate Kibana HTTPS Certificate Using OpenSSL
* Appendix B – Generate Kibana HTTPS Certificate Using Elasticsearch Certutil

---

## 2. Lab Environment

| Hostname    | IP Address    | Role          |
| ----------- | ------------- | ------------- |
| es-node-1   | 192.168.0.124 | Elasticsearch |
| es-node-2   | 192.168.0.125 | Elasticsearch |
| es-node-3   | 192.168.0.126 | Elasticsearch |
| kibana-node | 192.168.0.123 | Kibana        |

---

## 3. Target Architecture

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

## 4. TLS Trust Model

Understanding certificate trust is critical.

### Browser → Kibana

Kibana acts as a server.

Requirements:

* Kibana presents a server certificate.
* Browser validates the Kibana certificate.
* Browser trusts the CA that signed the Kibana certificate.

---

### Kibana → Elasticsearch

Kibana acts as a client.

Requirements:

* Elasticsearch presents a server certificate.
* Kibana validates the Elasticsearch certificate.
* Kibana trusts the Elasticsearch CA.

---

### TLS Role Summary

| Component     | Role            | Server Certificate | CA Trust Required |
| ------------- | --------------- | ------------------ | ----------------- |
| Browser       | Client          | No                 | Kibana CA         |
| Kibana        | Server + Client | Yes                | Elasticsearch CA  |
| Elasticsearch | Server          | Yes                | No                |

---

## Elasticsearch Prerequisites

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

## Kibana Certificate Directory Structure

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

## Configure Elasticsearch CA Trust

Kibana must trust the **CA** that **signed Elasticsearch HTTP** certificates.

#

### Copy Elasticsearch CA Certificate

Copy the CA certificate to Kibana.

- Example:

  ```bash
  scp /etc/elasticsearch/certs/zipcert/kibana/elasticsearch-ca.pem \
  root@192.168.0.123:/etc/kibana/certs/elastic-certs/
  ```

Only the **public CA certificate** should be copied.

Never copy:

* CA private keys
* Elasticsearch private keys
* PKCS12 bundles containing private keys

Note:

- Remmeber this **certificate was generated** during the elastic search **https certificate** generation.


#

If somehow Mentioned step was missing during certificate generation, then we can extract `elasticsearch-ca.pem` certificate from `elastic-stack-ca.p12` bundle.

### Extract `elasticsearch-ca.pem` File

- Run on `es-node-1`:

  ```bash
  openssl pkcs12 \
  -in /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
  -cacerts -nokeys \
  -out /etc/elasticsearch/certs/ca/elasticsearch-ca.pem
  ```

- Explanation:

  - cacerts → export certificates only
  - nokeys → NEVER export private keys

- It will ask:
  
  ```
  Enter Import Password:
  ```

- Provide the password used when creating the CA.

  ```text
  ElasticCA@123
  ```

#

### Copy ONLY CA Public Certificate to Kibana

```bash
scp /etc/elasticsearch/certs/ca/elasticsearch-ca.pem \
root@192.168.0.123:/etc/kibana/certs/elastic-certs/
```

Now Kibana receives ONLY:
- public CA certificate
- no secret material

---

## Verify CA Certificate

```bash
openssl x509 \
-in /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem \
-text -noout
```

Verify:

* Subject
* Issuer
* Validity dates
* Public key information

No private key should exist.


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

---

## Validate Permissions

```bash
namei -l /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem
```

Verify the kibana group can access the certificate.

---

# 8. Browser Trust Requirements

For browsers to trust Kibana HTTPS, they must trust the CA that signed the Kibana server certificate.

---

## Windows

Import:

```text
kibana-ca.crt
```

Into:

```text
Trusted Root Certification Authorities
```

---

## RHEL / Rocky / AlmaLinux

```bash
cp kibana-ca.crt \
/etc/pki/ca-trust/source/anchors/
```

Update trust:

```bash
update-ca-trust
```

---

## Ubuntu

```bash
cp kibana-ca.crt \
/usr/local/share/ca-certificates/
```

```bash
update-ca-certificates
```

---

# 9. Create Kibana Authentication Credentials

Kibana requires authentication to communicate with Elasticsearch.

---

## Reset kibana_system Password

Run on any Elasticsearch node.

```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password \
-u kibana_system
```

Example:

```text
New value: XXXXXXXXXXXXX
```

Store this password securely.

---

# 10. Create Kibana Keystore

Sensitive values should never be stored directly inside `kibana.yml`.

---

## Create Keystore

```bash
/usr/share/kibana/bin/kibana-keystore create
```

---

## Add Kibana Password

```bash
/usr/share/kibana/bin/kibana-keystore add elasticsearch.password
```

Enter:

```text
kibana_system password
```

---

## Verify Keystore Exists

```bash
ls -l /etc/kibana/
```

Expected:

```text
kibana.keystore
```

---

# 11. Generate Kibana Encryption Keys

Kibana requires encryption keys for secure operation.

These keys must remain constant across restarts.

---

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

# 12. Configure Kibana

Edit:

```bash
vi /etc/kibana/kibana.yml
```

---

## Kibana Server Configuration

```yaml
server.port: 5601

server.host: "0.0.0.0"

server.name: "kibana-node"
```

---

## Enable HTTPS

```yaml
server.ssl.enabled: true

server.ssl.certificate: /etc/kibana/certs/kibana-certs/kibana-server.crt

server.ssl.key: /etc/kibana/certs/kibana-certs/kibana-server.key
```

---

## Elasticsearch Connection

```yaml
elasticsearch.hosts:
  - "https://es-node-1:9200"
  - "https://es-node-2:9200"
  - "https://es-node-3:9200"
```

---

## Authentication

```yaml
elasticsearch.username: "kibana_system"
```

Password is stored in:

```text
kibana keystore
```

---

## Elasticsearch CA Trust

```yaml
elasticsearch.ssl.certificateAuthorities:
  - /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem
```

---

## Certificate Verification

```yaml
elasticsearch.ssl.verificationMode: full
```

`full` verifies:

* Certificate validity
* CA trust
* Hostname validation

---

## Kibana Encryption Keys

```yaml
xpack.security.encryptionKey: "<generated-key>"

xpack.encryptedSavedObjects.encryptionKey: "<generated-key>"

xpack.reporting.encryptionKey: "<generated-key>"
```

---

# 13. DNS Resolution Validation

Kibana must resolve Elasticsearch hostnames.

Verify:

```bash
getent hosts es-node-1
getent hosts es-node-2
getent hosts es-node-3
```

Expected:

```text
192.168.0.124 es-node-1
192.168.0.125 es-node-2
192.168.0.126 es-node-3
```

---

# 14. TLS Validation Before Startup

Validate Elasticsearch certificate trust.

---

## Verify Elasticsearch HTTPS

```bash
curl \
--cacert /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem \
-u elastic \
https://es-node-1:9200
```

Expected:

Successful response without certificate errors.

---

## Verify TLS Handshake

```bash
openssl s_client \
-connect es-node-1:9200 \
-CAfile /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem
```

Expected:

```text
Verify return code: 0 (ok)
```

---

## Verify Kibana Certificate

```bash
openssl x509 \
-in /etc/kibana/certs/kibana-certs/kibana-server.crt \
-text -noout
```

Verify:

* Subject
* Issuer
* SAN entries
* Expiration date

---

# 15. Firewall Configuration

## Kibana Node

```bash
firewall-cmd --permanent --add-port=5601/tcp
```

```bash
firewall-cmd --reload
```

---

## Elasticsearch Nodes

```bash
firewall-cmd --permanent --add-port=9200/tcp
```

```bash
firewall-cmd --reload
```

---

# 16. Start Kibana Service

Reload systemd:

```bash
systemctl daemon-reload
```

Enable service:

```bash
systemctl enable kibana
```

Start service:

```bash
systemctl start kibana
```

---

# 17. Verify Service Status

```bash
systemctl status kibana
```

Expected:

```text
active (running)
```

---

# 18. Verify Listening Port

```bash
ss -tlnp | grep 5601
```

Expected:

```text
LISTEN 0 511 0.0.0.0:5601
```

---

# 19. Verify Kibana Health API

```bash
curl -k https://kibana-node:5601/api/status
```

Expected:

```json
{
  "overall": {
    "level": "available"
  }
}
```

---

# 20. Monitor Kibana Logs

```bash
journalctl -u kibana -f
```

Successful startup should show:

```text
Kibana is now available
```

---

# 21. Browser Access Verification

Open:

```text
https://kibana-node:5601
```

Verify:

* HTTPS lock icon
* No certificate warnings
* Kibana login page loads

---

# 22. Verify Kibana ↔ Elasticsearch Integration

Login to Kibana.

Verify access to:

* Stack Management
* Dev Tools
* Index Management
* Monitoring

---

## Dev Tools Validation

Run:

```json
GET _cluster/health
```

Expected:

```json
{
  "status": "green"
}
```

---

# 23. Common Troubleshooting

| Problem                            | Possible Cause                   |
| ---------------------------------- | -------------------------------- |
| Kibana won't start                 | Certificate permission issue     |
| Certificate warning in browser     | CA not trusted                   |
| Unable to connect to Elasticsearch | Wrong CA path                    |
| Authentication failed              | Incorrect kibana_system password |
| TLS handshake failure              | SAN mismatch                     |
| Hostname verification failure      | DNS issue                        |
| Connection timeout                 | Firewall issue                   |

---

# 24. Production Recommendations

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

# 25. Summary

This SOP configured:

* Kibana HTTPS
* Elasticsearch CA trust
* Kibana authentication
* Kibana keystore
* Encryption keys
* TLS validation
* DNS validation
* Firewall configuration
* Kibana startup verification
* Kibana ↔ Elasticsearch integration validation

The next documents in this series are:

* **Appendix A – Generate Kibana HTTPS Certificate Using OpenSSL**
* **Appendix B – Generate Kibana HTTPS Certificate Using Elasticsearch Certutil**
