In this guide we will use native built in elasticsearch certificate generation tools.

## Lab Envrionment

Elasticsearch Node Inoformation:

| Hostname | IP            |
| -------- | ------------- |
| es-node-1 | 192.168.10.11 |
| es-node-2 | 192.168.10.12 |
| es-node-3 | 192.168.10.13 |

---

## Elasticsearch 8 Security

From Elasticsearch 8:

* Security is enabled by default
* TLS can be auto-generated
* Built-in users exist

But for LEARNING and REAL ADMINISTRATION, we should learn **MANUAL TLS configuration**.

That is what we will do.

---

## Elasticsearch Certificate Tools

Elasticsearch provides:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil
```

This is the official certificate generation tool.

You will use it for:

* CA generation
* Node certificates
* HTTP certificates

---

## Now Understand the Architecture

Elasticsearch uses **TWO different** network layers.

1. Transport Layer (Node-to-Node Communication)
2. HTTP Layer (Client-to-Elasticsearch)

### Transport TLS and HTTP TLS are DIFFERENT

| Type          | Used For       | Port |
| ------------- | -------------- | ---- |
| Transport TLS | Node-to-node   | 9300 |
| HTTP TLS      | Client-to-node | 9200 |

---

## Certificate Basics

Before configuration, understand certificates properly.


### What is a Certificate?

A certificate is an identity card.

It proves:

* "I am node1"
* "I am trusted"
* "I was signed by a trusted CA"


### What is a CA (Certificate Authority)?

* CA = Trusted signer.
* The CA signs node certificates.

---

## Recommended Configuration Order

We will configure in this order:

1. Create CA
2. Create transport certificates
3. Configure transport TLS
4. Form secure cluster
5. Create HTTP certificates
6. Configure HTTPS
7. Configure Kibana/Filebeat later

---

## Directory Structure

Certificates Genration important paths:

| Purpose                          | Path                                 |
| -------------------------------- | ------------------------------------ |
| Config                           | `/etc/elasticsearch/`                |
| Root Certificate Directory       | `/etc/elasticsearch/certs/`          |
| CA Certificates                  | `/etc/elasticsearch/certs/ca/`       |
| Elasticsearch Node Certificates  | `/etc/elasticsearch/certs/node/`     |
| HTTP Certificates                | `/etc/elasticsearch/certs/http/`     |
| Kibana Certificates *(future)*   | `/etc/elasticsearch/certs/kibana/`   |
| Logstash Certificates *(future)* | `/etc/elasticsearch/certs/logstash/` |
| Filebeat Certificates *(future)* | `/etc/elasticsearch/certs/filebeat/` |
| Fleet Certificates *(future)*    | `/etc/elasticsearch/certs/fleet/`    |
| APM Certificates *(future)*      | `/etc/elasticsearch/certs/apm/`      |


Directory Layout:

```text
/etc/elasticsearch/certs/
├── ca/
├── node/
├── http/
├── kibana/
├── logstash/
├── filebeat/
├── fleet/
└── apm/
```

---

## PHASE 1 — Create Certificate Authority (CA)

You can create certificates:

* on es-node-1
* or separate admin machine

Recommended: Use es-node-1.

#

### Step 1: Create Certificate Directory

On node1:

```bash
mkdir -p /etc/elasticsearch/certs
```

Set ownership:

```bash
chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```

Set permission:

```bash
chmod 750 /etc/elasticsearch/certs
```

#

### Step 2: Generate CA

Run:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil ca
```

You will see:

```text
Please enter the desired output file
```

Press ENTER.

Default:

```text
elastic-stack-ca.p12
```

Then:

```text
Enter password for CA
```

Choose strong password.

Example:

```text
ElasticCA@123
```

#

### What Was Created?

A file:

```bash
elastic-stack-ca.p12
```

This contains:

* CA certificate
* CA private key

#

### Move CA File

Move it:

```bash
mv elastic-stack-ca.p12 /etc/elasticsearch/certs/
```

Set permission:

```bash
chown elasticsearch:elasticsearch /etc/elasticsearch/certs/elastic-stack-ca.p12
```

---

## PHASE 2 — Generate Transport Certificates

Now we create certificates for:

* es-node-1
* es-node-2
* es-node-3

These certificates secure:

```text
9300 transport communication
```

#

### Method Choices

There are 2 methods:

| Method                        | Description              |
| ----------------------------- | ------------------------ |
| One certificate for all nodes | Easier lab               |
| Separate certificate per node | Production best practice |

we will use: **One certificate for all nodes**

#

### Step 1: Create instances.yml

Create file:

```bash
vim /etc/elasticsearch/certs/instances.yml
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

#

### Why This File?

This defines:

* certificate names
* hostnames
* IP addresses

These become:

* SANs (Subject Alternative Names)


#

### Step 2: Generate Node Certificates

Run:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
--ca /etc/elasticsearch/certs/elastic-stack-ca.p12 \
--in /etc/elasticsearch/certs/instances.yml \
--out /etc/elasticsearch/certs/elastic-certificates.p12
```

It asks:

```text
Enter password for CA
```

Provide CA password.

Then asks:

```text
Enter password for elastic-certificates.p12
```

Choose password.

Example:

```text
ElasticNode@123
```

#

### What Was Created?

```bash
elastic-certificates.p12
```

Contains:

* node certificates
* private keys
* CA cert

#

### Important Learning Point

This `.p12` is a PKCS#12 keystore.

It bundles:

* certificates
* private keys
* CA chain

into one encrypted file.

---

## PHASE 3 — Distribute Certificates

### Copy the certificate file to ALL nodes.

On node1:

```bash
scp /etc/elasticsearch/certs/elastic-certificates.p12 root@192.168.10.12:/etc/elasticsearch/certs/
```

```bash
scp /etc/elasticsearch/certs/elastic-certificates.p12 root@192.168.10.13:/etc/elasticsearch/certs/
```

Also copy:

```bash
elastic-stack-ca.p12
```

>optional but recommended

#

### Set Permissions on ALL Nodes

On every node:

```bash
chown elasticsearch:elasticsearch /etc/elasticsearch/certs/*
```

```bash
chmod 640 /etc/elasticsearch/certs/*
```

---

## PHASE 4 — Configure HTTP TLS

This secures:

```text
9200 HTTPS API
```

#

### Generate HTTP Certificates

Run:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil http
```

This is interactive.

#

### Answers During Wizard

Choose carefully:


Generate CSR?

```text
n
```

Reason:
We use our own CA.



Use Existing CA?

```text
y
```

CA Path

```text
/etc/elasticsearch/certs/elastic-stack-ca.p12
```


Enter CA Password

```text
Provide CA password.
```

Validity

Example:

```text
3650D
```

10 years.


One certificate per node?

```text
y
```

Enter node names and IPs

Same as earlier:

* es-node-1
* es-node-2
* es-node-3

#

### Output

Creates:

```bash
elasticsearch-ssl-http.zip
```

Extract:

```bash
unzip elasticsearch-ssl-http.zip
```

You will get:

* http.p12
* sample configs

#

### Copy HTTP Certificates

Place appropriate file on each node:

Example:

```bash
/etc/elasticsearch/certs/http.p12
```
