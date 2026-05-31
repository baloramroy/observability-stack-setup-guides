# Generate Kibana HTTPS Certificate Using Elasticsearch Certutil

## Purpose

This SOP explains how to generate a **Kibana HTTPS certificate** using **Elasticsearch `certutil`**, which is the **recommended enterprise method** when working inside the **Elastic Stack ecosystem**.

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
* The Elastic Stack CA (elastic-stack-ca.p12) must already exist.
  
  ```
  This CA was previously created during Elasticsearch TLS certificate generation.
  ```

* `Kibana` is installed
* Kibana server `IP and hostname` are known

---

## Create Working Directory

- Run on elasticsearch-node-1 (certificate management node):

  ```bash
  mkdir -p /root/kibana-certutil
  cd /root/kibana-certutil
  ```

---

## Create Kibana Instance Definition File

- This file defines all Kibana endpoints and SAN entries.

  ```bash
  vim kibana-instance.yml
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

- Run:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --in /root/kibana-certutil/kibana-instance.yml \
  --ca /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
  --out /root/kibana-certutil/kibana-cert.zip
  ```

- If CA is password protected, then enter CA pass:

  ```bash
  ElasticCA@123
  ```

- Enter password for kibana-cert/kibana-cert.p12:
  
  ```bash
  KibanaHttp@123
  ```
> Then a zip file will generate for Kibana Certificate.


---

## Extract Generated Certificate

- Unzip:

  ```bash
  unzip kibana-cert.zip
  ```

- Expected structure:

  ```text
  kibana-cert/
  └── kibana-cert.p12
  ```

## Validate PKCS12 Contents

- The resulting .p12 file often contains:

  - The Kibana **server certificate**
  - The Kibana **private key**
  - The **CA certificate** (sometimes included)

- To check run:

  ```bash
  openssl pkcs12 \
  -in /root/kibana-certutil/kibana-cert/kibana-cert.p12 \
  -info -nodes
  ```

- Verify:
    - Certificate
    - Private Key
    - CA Certificate
  
---

## Convert PKCS12 to PEM

Kibana prefer PEM format (recommended for Kibana SOP consistency):

- Run this to only extract certificate in pem encoded 
  ```bash
  openssl pkcs12 \
  -in /root/kibana-certutil/kibana-cert/kibana-cert.p12 \
  -clcerts -nokeys \
  -out kibana-server.crt
  ```

- Run this to only extract the private key

  ```bash
  openssl pkcs12 \
  -in /root/kibana-certutil/kibana-cert/kibana-cert.p12 \
  -nocerts -nodes \
  -out kibana-server.key
  ```

---

## Verify Certificate Details

- Run:

  ```bash
  openssl x509 -in /root/kibana-certutil/kibana-cert/kibana-server.crt -text -noout
  ```

- This verifies:

  - Subject
  - Issuer
  - SAN
  - Expiry
  - Key Usage

---

## Verify Certificate and Private Key Match

- Certificate:

  ```bash
  openssl x509 -noout -modulus -in /root/kibana-certutil/kibana-cert/kibana-server.crt | openssl md5
  ```

- Private Key:
  ```bash
  openssl rsa -noout -modulus -in /root/kibana-certutil/kibana-cert/kibana-server.key | openssl md5
  ```

- The hashes must match.

  ```text
  MD5(stdin)= a1b2c3d4...
  MD5(stdin)= a1b2c3d4...
  ```
> If they differ, Kibana will fail to start.

---

## Certificate Deliverables

Example:

- At the end of this **procedure** the following **files** should exist:

  - kibana-server.crt
  - kibana-server.key

- If PKCS12 is retained:

  - kibana-cert.p12
  - kibana-server.crt
  - kibana-server.key


---

## Certificate Validation Checklist

Verify:

- ✓ Certificate contains correct hostname
- ✓ Certificate contains correct IP address
- ✓ SAN entries are present
- ✓ Certificate signed by Elastic Stack CA
- ✓ Certificate and private key match
- ✓ Expiration date is acceptable

---

## Secure Storage Recommendation

- Store the following securely:

    - kibana-server.key
    - kibana-cert.p12
    - elastic-stack-ca.p12

- Never store private keys in source repositories.
- Restrict access to authorized administrators only.

---

## Security Advantages of Certutil Method

* Uses Elastic-native PKI
* No manual OpenSSL complexity
* Single CA for entire stack
* Easier renewal process
* Lower human error risk
* Production recommended

---

## Comparison Summary (OpenSSL vs Certutil)

| Feature               | OpenSSL | Certutil |
| --------------------- | ------- | -------- |
| Control               | High    | Medium   |
| Complexity            | High    | Low      |
| Elastic compatibility | Medium  | High     |
| Recommended for ELK   | No      | Yes      |
| Production usage      | Rare    | Common   |

---

## Appendix Completion Summary

This appendix successfully:

* Used Elasticsearch CA (`elastic-stack-ca`)
* Generated Kibana certificate via certutil
* Ensured SAN correctness
* Converted to PEM format
* Prepared Kibana deployment structure
* Integrated with secure Kibana TLS architecture

---
