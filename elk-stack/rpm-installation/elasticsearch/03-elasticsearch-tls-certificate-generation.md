# Elasticsearch TLS Certificate Generation

## Scope of This SOP

This SOP focuses only on **Elasticsearch TLS certificate generation** using the native built-in **Elasticsearch certificate utility**.

This document covers:

* **Certificate Authority (CA)** generation
* **Transport layer TLS** certificate generation
* **HTTP layer TLS** certificate generation

This document does **NOT** cover:

* Elasticsearch TLS configuration
* `elasticsearch.yml`
* Cluster formation
* Security configuration
* Kibana integration
* Logstash integration
* Filebeat/Fleet/APM integration

These topics will be covered in separate SOPs.

---

## Lab Environment

### Elasticsearch Node Information

| Hostname  | IP            |
| --------- | ------------- |
| es-node-1 | 192.168.10.11 |
| es-node-2 | 192.168.10.12 |
| es-node-3 | 192.168.10.13 |

---

## Elasticsearch 8 Security

From Elasticsearch 8:

* Security is **enabled by default**
* TLS can be **auto-generated**
* **Built-in users** exist

However, for learning and real administration, it is important to understand **manual TLS certificate generation** and management.

---

## Elasticsearch Certificate Tool

Elasticsearch provides the built-in certificate utility:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil
```

This tool is used for:

* CA generation
* Transport TLS certificate generation
* HTTP TLS certificate generation

---

## Elasticsearch TLS Architecture

Elasticsearch uses two separate network layers.


### Transport Layer TLS

Used for secure **node-to-node communication**.

| Type          | Used For                   | Port |
| ------------- | -------------------------- | ---- |
| Transport TLS | Node-to-node communication | 9300 |

#

### HTTP Layer TLS

Used for secure **client/API communication**.

| Type     | Used For                     | Port |
| -------- | ---------------------------- | ---- |
| HTTP TLS | Client-to-node communication | 9200 |

---

## Certificate Basics

### What is a Certificate?

A certificate acts like a digital identity card.

It proves:

* The identity of the node or service
* The node/service is trusted
* The certificate was signed by a trusted CA

#

## What is a CA (Certificate Authority)?

A Certificate Authority (CA) is a trusted signer.

The CA signs certificates for Elasticsearch nodes and services.

---

## Certificate Directory Structure

### Important Certificate Paths

| Purpose                               | Path                                  |
| ------------------------------------- | ------------------------------------- |
| Elasticsearch Configuration Directory | `/etc/elasticsearch/`                 |
| Root Certificate Directory            | `/etc/elasticsearch/certs/`           |
| CA Certificates                       | `/etc/elasticsearch/certs/ca/`        |
| Transport Layer Certificates          | `/etc/elasticsearch/certs/transport/` |
| HTTP Layer Certificates               | `/etc/elasticsearch/certs/http/`      |

#

### Directory Layout

```text
/etc/elasticsearch/certs/
├── ca/
├── transport/
└── http/
```

---

## Transport Layer TLS Certificate Generation

Transport TLS secures Elasticsearch **node-to-node communication** on port `9300`.


### Create Certificate Directories

Run on `es-node-1`:

```bash
mkdir -p /etc/elasticsearch/certs/{ca,transport,http}
```

Set Ownership

```bash
chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```

Set Permissions

```bash
chmod -R 750 /etc/elasticsearch/certs
```

#

### Create Certificate Authority (CA)

The CA will be used to sign Elasticsearch certificates.

Recommended location:

* `es-node-1`
* or a dedicated administration machine

#

**Step 1 — Generate CA**

Run:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil ca
```

**Step 2 — Output File**

When prompted:

```text
Please enter the desired output file
```

Enter:

```text
/etc/elasticsearch/certs/ca/elastic-stack-ca.p12
```


**Step 3 — Set CA Password**

Example:

```text
ElasticCA@123
```

Use a strong password in production environments.

#

### Generated CA File

Generated file:

```bash
/etc/elasticsearch/certs/ca/elastic-stack-ca.p12
```

This **PKCS#12** file contains:

* CA certificate
* CA private key

---

## Generate Transport Layer Certificates

These certificates secure:

```text
9300 transport communication
```

#

### Certificate Generation Methods

| Method                        | Description              |
| ----------------------------- | ------------------------ |
| One certificate for all nodes | Easier for labs          |
| Separate certificate per node | Production best practice |

This SOP uses:

```text
One certificate for all nodes
```

#

### Step 1 — Create instances.yml

Create file:

```bash
vim /etc/elasticsearch/certs/transport/instances.yml
```

Content:

```yaml
instances:
  - name: es-node-1
    dns:
      - es-node-1
    ip:
      - 192.168.10.11

  - name: es-node-2
    dns:
      - es-node-2
    ip:
      - 192.168.10.12

  - name: es-node-3
    dns:
      - es-node-3
    ip:
      - 192.168.10.13
```


### Why instances.yml is Required

This file defines:

* Certificate names
* DNS names
* IP addresses

These become:

* SANs (Subject Alternative Names)

inside the certificates.

#

### Step 2 — Generate Transport Certificates

**Run:**

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
--ca /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
--in /etc/elasticsearch/certs/transport/instances.yml \
--out /etc/elasticsearch/certs/transport/elastic-transport-certificates.p12
```


**Provide Passwords**

You will be prompted for:

- CA Password

  Example:

  ```text
  ElasticCA@123
  ```

- Transport Certificate Password

  Example:

  ```text
  ElasticTransport@123
  ```

#

## Generated Transport Certificate File

Generated file:

```bash
/etc/elasticsearch/certs/transport/elastic-transport-certificates.p12
```

This **PKCS#12** keystore contains:

* Node certificates
* Private keys
* CA certificate chain

---

## Distribute Transport Certificates

Copy certificates to all Elasticsearch nodes.


### Copy Transport Certificate

From `es-node-1`:

```bash
scp /etc/elasticsearch/certs/transport/elastic-transport-certificates.p12 \
root@192.168.10.12:/etc/elasticsearch/certs/transport/
```

```bash
scp /etc/elasticsearch/certs/transport/elastic-transport-certificates.p12 \
root@192.168.10.13:/etc/elasticsearch/certs/transport/
```

#

### Copy CA Certificate

Recommended:

```bash
scp /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
root@192.168.10.12:/etc/elasticsearch/certs/ca/
```

```bash
scp /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
root@192.168.10.13:/etc/elasticsearch/certs/ca/
```

---

## Set Certificate Permissions

Run on all Elasticsearch nodes.


### Set Ownership

```bash
chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```


### Set Permissions

```bash
chmod -R 640 /etc/elasticsearch/certs/*
```

---

## HTTP Layer TLS Certificate Generation

HTTP TLS secures **Elasticsearch API communication** on port `9200`.


### Generate HTTP Certificates

Run:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil http
```

This command starts an interactive wizard.

#

### HTTP Certificate Wizard Answers

**Generate CSR?**

- Choose:

  ```text
  n
  ```

- Reason:

  ```text
  We are using our own internal CA.
  ```
#

**Use Existing CA?**

- Choose:

  ```text
  y
  ```

#

**Enter CA Path**

- Paste

  ```text
  /etc/elasticsearch/certs/ca/elastic-stack-ca.p12
  ```

#

**Enter CA Password**

- Provide the CA password.

  Example:

  ```text
  ElasticCA@123
  ```

#

**Certificate Validity**

- Example:

  ```text
  3650D
  ```

- This means:

  ```text
  10 years
  ```

#

**One Certificate Per Node?**

- Choose:

  ```text
  y
  ```

#

**Enter Node Information**

Provide:

* es-node-1
* es-node-2
* es-node-3

along with their IP addresses.

---

## Generated HTTP Certificate Package

Generated file:

```bash
elasticsearch-ssl-http.zip
```

Extract HTTP Certificate Package

```bash
unzip elasticsearch-ssl-http.zip
```

The archive contains:

  * HTTP certificates
  * PKCS#12 keystores
  * Sample configuration files


After extraction, Elasticsearch creates node-specific HTTP certificates.
```
/elasticsearch/
├── es-node-1/
│   └── http.p12
├── es-node-2/
│   └── http.p12
└── es-node-3/
    └── http.p12
```

---

## Copy HTTP Certificates

Place appropriate HTTP certificate files into:

```bash
/etc/elasticsearch/certs/http/
```

Example:

```bash
/etc/elasticsearch/certs/http/http.p12
```

---

## Important Difference

### Transport TLS
You used:
```
One certificate for all nodes
```

So:
- same .p12 file
- goes to every node.

### HTTP TLS
You selected:
```
One certificate per node = YES
```

So:
- each node gets different http.p12


---

## Summary

This SOP covered:

* CA generation
* Transport layer TLS certificate generation
* HTTP layer TLS certificate generation

TLS **configuration** and Elasticsearch **security integration** will be covered in separate SOPs.
