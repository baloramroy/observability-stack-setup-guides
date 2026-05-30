# Appendix B – Generate Kibana HTTPS Certificate Using Elasticsearch Certutil

## Purpose of This Appendix

This appendix explains how to generate a **Kibana HTTPS certificate** using **Elasticsearch `certutil`**, which is the **recommended enterprise method** when working inside the **Elastic Stack ecosystem**.

This method ensures:

* Consistent PKI across Elasticsearch and Kibana
* Simplified certificate lifecycle management
* Proper SAN handling
* Easier renewal and automation

After completion:

```text
Browser → HTTPS → Kibana (certutil-generated TLS)
```

---

## Why Use Elasticsearch Certutil

- Using `elasticsearch-certutil` provides advantages over OpenSSL:

    | Feature               | OpenSSL       | Certutil |
    | --------------------- | ------------- | -------- |
    | Elastic compatibility | Manual        | Native   |
    | SAN handling          | Manual config | Built-in |
    | Automation            | Hard          | Easy     |
    | Consistency           | Medium        | High     |
    | Recommended for ELK   | No            | Yes      |

---

## Architecture Overview

- We will use the **existing Elastic Stack CA** for:

  ```text
  elastic-stack-ca.p12
      ├── Elasticsearch HTTP/Transport TLS
      └── Kibana HTTPS certificate
  ```

> No separate Kibana CA is required.

---

## Prerequisites

Before proceeding ensure:

* `Elasticsearch cluster` is installed
* `elasticsearch-certutil` is available
* `CA` already exists:

  ```text
  elastic-stack-ca.p12
  ```

* `Kibana` is installed
* Kibana server `IP and hostname` are known

---

## Create Working Directory

Run on any Elasticsearch node (recommended: es-node-1):

```bash
mkdir -p /root/kibana-certutil
cd /root/kibana-certutil
```

---

## Create Kibana Instance Definition File

- This file defines all Kibana endpoints and SAN entries.

  ```bash
  vi kibana-instance.yml
  ```


- Example Configuration

  ```yaml
  instances:
    - name: kibana-node
      dns:
        - kibana-node
        - kibana.local
      ip:
        - 192.168.0.123
  ```

- Explanation

  | Field | Purpose                     |
  | ----- | --------------------------- |
  | name  | Kibana certificate identity |
  | dns   | DNS names used by browser   |
  | ip    | Direct IP access            |

>⚠️ Every access method must be included here.

---

## Generate Kibana Certificate Using Certutil

Run:

```bash id="f5r9wa"
elasticsearch-certutil cert \
--in kibana-instance.yml \
--ca /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
--ca-pass "" \
--out kibana-cert.zip
```

---

## If CA Password Exists

If CA is password protected:

```bash id="g6t2yb"
--ca-pass "YOUR_CA_PASSWORD"
```

---

# 8. Extract Generated Certificate

Unzip:

```bash id="h8u1zc"
unzip kibana-cert.zip
```

Expected structure:

```text id="i4v9ld"
kibana/
 ├── kibana-node.crt
 ├── kibana-node.key
 └── kibana-node.p12
```

---

# 9. Convert PKCS12 (Optional but Recommended)

If you prefer PEM format (recommended for Kibana SOP consistency):

```bash id="j1x6mf"
openssl pkcs12 \
-in kibana/kibana-node.p12 \
-clcerts -nokeys \
-out kibana-server.crt
```

```bash id="k3z8qn"
openssl pkcs12 \
-in kibana/kibana-node.p12 \
-nocerts -nodes \
-out kibana-server.key
```

---

## Verify Conversion

```bash id="l5p0wr"
openssl x509 -in kibana-server.crt -text -noout
```

---

# 10. Validate Certificate Chain

```bash id="m7q2ts"
openssl pkcs12 \
-in kibana/kibana-node.p12 \
-info -nodes
```

Verify:

* Issuer = elastic-stack-ca
* Validity period
* Private key exists (for .p12 only)

---

# 11. Verify Certificate SAN

```bash id="n9r4uv"
openssl x509 \
-in kibana-server.crt \
-text -noout | grep -A1 "Subject Alternative Name"
```

Expected:

```text id="o1s6wx"
DNS:kibana-node
DNS:kibana.example.local
IP Address:192.168.0.123
```

---

# 12. Deploy Certificate to Kibana Server

Create directory:

```bash id="p3t8yz"
mkdir -p /etc/kibana/certs/kibana-certs
```

Copy files:

```bash id="q5u1ab"
scp kibana-server.crt root@192.168.0.123:/etc/kibana/certs/kibana-certs/

scp kibana-server.key root@192.168.0.123:/etc/kibana/certs/kibana-certs/
```

---

# 13. Set Ownership

On Kibana node:

```bash id="r7v3cd"
chown -R root:kibana /etc/kibana/certs
```

---

# 14. Set Permissions

```bash id="s9w5ef"
find /etc/kibana/certs -type d -exec chmod 750 {} \;
find /etc/kibana/certs -type f -exec chmod 640 {} \;
```

---

# 15. Validate Certificate on Kibana Node

```bash id="t1x7gh"
openssl x509 \
-in /etc/kibana/certs/kibana-certs/kibana-server.crt \
-text -noout
```

Check:

* CN matches Kibana hostname
* SAN includes IP/DNS
* Issuer = elastic-stack-ca

---

# 16. Elasticsearch CA Trust (Important)

Since certutil uses Elastic CA, Kibana must trust it.

Copy CA:

```bash id="u3y9ij"
scp /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
root@192.168.0.123:/etc/kibana/certs/elastic-certs/
```

---

## Convert CA to PEM

```bash id="v5z1kl"
openssl pkcs12 \
-in elastic-stack-ca.p12 \
-cacerts -nokeys \
-out elasticsearch-ca.pem
```

---

## Verify CA

```bash id="w7a3mn"
openssl x509 -in elasticsearch-ca.pem -text -noout
```

---

# 17. Final Kibana Certificate Layout

```text id="x9b5op"
/etc/kibana/certs/

├── elastic-certs/
│   └── elasticsearch-ca.pem
│
└── kibana-certs/
    ├── kibana-server.crt
    └── kibana-server.key
```

---

# 18. Kibana Configuration Reminder

In `kibana.yml`:

```yaml id="y1c7qr"
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana-certs/kibana-server.crt
server.ssl.key: /etc/kibana/certs/kibana-certs/kibana-server.key

elasticsearch.ssl.certificateAuthorities:
  - /etc/kibana/certs/elastic-certs/elasticsearch-ca.pem

elasticsearch.ssl.verificationMode: full
```

---

# 19. Security Advantages of Certutil Method

* Uses Elastic-native PKI
* No manual OpenSSL complexity
* Single CA for entire stack
* Easier renewal process
* Lower human error risk
* Production recommended

---

# 20. Comparison Summary (OpenSSL vs Certutil)

| Feature               | OpenSSL | Certutil |
| --------------------- | ------- | -------- |
| Control               | High    | Medium   |
| Complexity            | High    | Low      |
| Elastic compatibility | Medium  | High     |
| Recommended for ELK   | No      | Yes      |
| Production usage      | Rare    | Common   |

---

# 21. Appendix Completion Summary

This appendix successfully:

* Used Elasticsearch CA (`elastic-stack-ca`)
* Generated Kibana certificate via certutil
* Ensured SAN correctness
* Converted to PEM format
* Prepared Kibana deployment structure
* Integrated with secure Kibana TLS architecture

---

## Final Note

For enterprise ELK deployments:

👉 **Certutil method is the preferred standard**
👉 OpenSSL method is mainly for learning or external PKI integration environments
