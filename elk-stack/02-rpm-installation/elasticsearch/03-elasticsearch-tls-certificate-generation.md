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

Create Directory for generated zip file certificate
```bash
mkdir -p /etc/elasticsearch/certs/zipcert/{transport,http}
```

#

### Create Certificate Authority (CA)

The CA will be used to **sign Elasticsearch** certificates.

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

### Output CA File

Generated file:

```bash
/etc/elasticsearch/certs/ca/elastic-stack-ca.p12
```

This `PKCS#12 (.p12)` files are encrypted containers that may include:
- certificates
- private keys
- CA chains

> The generated CA becomes the **root trust authority** for the Elasticsearch environment.

#

### Backup CA Certificate

Backup CA files securely.
- Losing the CA private key may **prevent future certificate generation or renewal**.

Production CA private keys should ideally be stored:
- offline
- encrypted
- access-controlled
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

* **SANs (Subject Alternative Names)** - inside the certificates.
* **Modern TLS** validation checks **SAN entries** instead of **Common Name (CN)**.

If node IPs or DNS names are missing from SANs:
- TLS validation fails
- nodes cannot trust each other

#

### Step 2 — Generate Transport Certificates

**Run:**

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
--ca /etc/elasticsearch/certs/ca/elastic-stack-ca.p12 \
--in /etc/elasticsearch/certs/transport/instances.yml \
--out /etc/elasticsearch/certs/zipcert/transport/elastic-transport-certificates.zip
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

## Unzip Generated Transport File

### Generated file:

```bash
/etc/elasticsearch/certs/zipcert/transport/elastic-transport-certificates.zip
```

>[!NOTE] 
This a zip file. So we have to unzip this file and distribute each certificate to each node.

### Extract bundle

```bash
unzip /etc/elasticsearch/certs/zipcert/transport/elastic-transport-certificates.zip
```

- Now you will get:

  ```
  es-node-1/es-node-1.p12
  es-node-2/es-node-2.p12
  es-node-3/es-node-3.p12
  ```

- This **PKCS#12** keystore contains:

  * Node certificates
  * Private keys
  * CA certificate chain

---

## Distribute Transport Certificates

Copy certificates to all Elasticsearch nodes.


### Copy Transport Certificate

From `es-node-1`:

```bash
scp -r /etc/elasticsearch/zipcert/transport/es-node-2 \
root@192.168.10.12:/etc/elasticsearch/certs/transport/
```

```bash
scp -r /etc/elasticsearch/zipcert/transport/es-node-3 \
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

For lab environments, copying the `CA PKCS#12` file is acceptable.

In production:
- keep CA private keys secured
- distribute only required trust certificates
- avoid exposing CA signing keys to cluster nodes

---

## Set Certificate Permissions

Run on all Elasticsearch nodes.


### Set Ownership

```bash
chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```


### Set Permissions

```bash
find /etc/elasticsearch/certs -type d -exec chmod 750 {} \;
find /etc/elasticsearch/certs -type f -exec chmod 640 {} \;
```

---

## HTTP Layer TLS Certificate Generation

HTTP TLS secures **Elasticsearch API communication** on port `9200`.


### Generate HTTP Certificates

Run on ONE Elasticsearch node (recommended: `es-node-1`).

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil http
```

This command starts an interactive wizard.

#


### HTTP Certificate Wizard Answers

**1. Generate CSR?**

- Choose:

  ```text
  n
  ```

- Example:

  ```text
  Generate a CSR? [y/N] n
  ```


**2. Use Existing CA?**

- Choose:

  ```text
  y
  ```

- Example:

  ```text
  Use an existing CA? [y/N] y
  ```


**3. Enter CA Path**

- Provide the CA `.p12` file path:

  ```text
  /etc/elasticsearch/certs/ca/elastic-stack-ca.p12
  ```

- Example:

  ```text
  CA Path: /etc/elasticsearch/certs/ca/elastic-stack-ca.p12
  ```


**4. Enter CA Password**

>[!Note] 
Provide the password used when the CA was created.

- Example:

   ```text
   Password for elastic-stack-ca.p12:
   ```

- Example password:

  ```text
  ElasticCA@123
  ```


#

### Certificate Validity

- Meaning:

  ```text
  3 years
  ```

- Example:

  ```text
  For how long should your certificate be valid? [5y] 3Y
  ```

- You may also use:

  | Value | Meaning |
  | ----- | ------- |
  | `90D` | 90 days |
  | `1Y`  | 1 year  |
  | `3Y`  | 3 years |
  | `5Y`  | 5 years |


#

### Generate One Certificate Per Node

- Choose:

  ```text
  y
  ```

- Example:

  ```text
  Generate a certificate per node? [y/N] y
  ```

---

### Node Information

**1. Node #1 Name**

- Example:

  ```text
  node #1 name: es-node-1
  ```


**2. Hostnames For es-node-1**

Enter **ALL DNS** names or **hostnames** clients may use to connect.

- Example:

  ```text
  es-node-1
  ```

  Press ENTER again when finished.

- Example:

  ```text
  You entered the following hostnames.

  - es-node-1

  Is this correct [Y/n] y
  ```

  > Do NOT enter IP addresses here.


**3. IP Addresses For es-node-1**

Enter the node IP address.

- Example:

  ```text
  192.168.0.124
  ```

  Press ENTER again when finished.

- Example:

  ```text
  You entered the following IP addresses.

  - 192.168.0.124

  Is this correct [Y/n] y
  ```

#

### Additional Certificate Options

- Recommended:

  ```text
  Do you wish to change any of these options? [y/N] n
  ```

- Default values are usually correct:

  ```text
  Key Size: 2048
  Key Usage: digitalSignature,keyEncipherment
  ```

#

### Generate Additional Certificates

- Choose:

  ```text
  y
  ```

  for remaining nodes.

- Example:

  ```text
  Generate additional certificates? [Y/n] y
  ```

#

### Node #2 Example

- Node Name

  ```text
  es-node-2
  ```

- Hostname

  ```text
  es-node-2
  ```

- IP Address

  ```text
  192.168.0.125
  ```

#

### Node #3 Example

- Node Name

  ```text
  es-node-3
  ```

- Hostname

  ```text
  es-node-3
  ```

- IP Address

  ```text
  192.168.0.126
  ```

#

### Finish Additional Certificates

- After the last node:

  ```text
  Generate additional certificates? [Y/n] n
  ```

#

### HTTP Certificate Password

This password protects the generated `http.p12` files.

- Example:

  ```text
  Provide a password for the "http.p12" file:
  ```

- Example password:

  ```text
  ElasticHttp@123
  ```

> You may leave it blank by pressing ENTER, but password protection is recommended.

#

### Output ZIP File Location

- Example:

  ```text
  What filename should be used for the output zip file?
  [/usr/share/elasticsearch/elasticsearch-ssl-http.zip]
  /etc/elasticsearch/certs/zipcert/http/elasticsearch-ssl-http.zip
  ```

- Final Output
  ```text
  Zip file written to:
  /etc/elasticsearch/certs/zipcert/http/elasticsearch-ssl-http.zip
  ```


---

## ZIP File Extraction

- Extract HTTP Certificate Package

  ```bash
  unzip /etc/elasticsearch/certs/zipcert/http/elasticsearch-ssl-http.zip
  ```

- The archive contains:

    * HTTP certificates
    * PKCS#12 keystores
    * Sample configuration files


- After extraction

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

After extraction, copy each node's `http.p12` file from each directory to its **corresponding Elasticsearch node.**

- Place appropriate HTTP certificate files into:

  ```bash
  /etc/elasticsearch/certs/http/
  ```

- Example:

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

## Certificate Renewal

The certificates we have generated:
- Eventually expire and require renewal.
- Renewal procedures should be planned before certificate expiration dates.

---

## Recommended Best Practice

For production Elasticsearch clusters:

1. Generate:

   * One certificate per node
   * Separate private key per node

2. Include:

   * Correct hostname
   * Correct node IP

3. Use:

   * Same CA for the whole cluster

This is the cleanest and most professional deployment model.

---

## Summary

This SOP covered:

* CA generation
* Transport layer TLS certificate generation
* HTTP layer TLS certificate generation

TLS **configuration** and Elasticsearch **security integration** will be covered in separate SOPs.

---

## Other Installation Guides:
- Previous: [Elasticsearch Installation Guide](02-elasticsearch-installation.md)
- Next: [Elasticsearch Post Configuration and Cluster Validation](04-elasticsearch-post-configuration-and-cluster-validation.md)
