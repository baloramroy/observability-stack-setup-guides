# Install Kibana on a Separate Node

## Background

This Kibana installation is part of an existing secure Elastic Stack deployment where:

- Elasticsearch cluster is already **installed and fully operational**
- **TLS/SSL** security is already enabled for:
  - Node-to-node communication (transport layer)
  - Client communication (HTTP layer)
- Certificates **(CA + node certificates)** are already generated and configured

---

## Key Objective of Kibana Setup:

Because Elasticsearch is already secured:

- Kibana MUST connect using HTTPS (not HTTP)
- Kibana MUST trust Elasticsearch CA certificate
- Authentication will be required (elastic user or service account token)
- Kibana setup will include TLS verification

⚠ This means Kibana installation is NOT standalone — it is tightly coupled with the existing secured Elasticsearch cluster.


---

## Lab Design (Kibana Node)

| VM  | Hostname    | IP (example)  | Role   |
| --- | ----------- | ------------- | ------ |
| VM4 | kibana-node | 192.168.0.123 | Kibana |

---

## Minimum System Requirements

| Resource | Minimum | Recommended |
| -------- | ------- | ----------- |
| CPU      | 1 vCPU  | 2 vCPU      |
| RAM      | 2 GB    | 4 GB        |
| Disk     | 20 GB   | 20–50 GB    |

Kibana is lightweight compared to Elasticsearch.


---

## OS Preparation (Kibana Node)


### Set Hostname

  ```bash
  hostnamectl set-hostname kibana-node
  ```

### Update System

  ```bash
  dnf update
  ```

### Configure `/etc/hosts`

- Edit:

  ```bash
  vi /etc/hosts
  ```

- Add all **ES nodes + Kibana** itself:

  ```text
  192.168.0.124  es-node-1
  192.168.0.125  es-node-2
  192.168.0.126  es-node-3
  192.168.0.123  kibana-node
  ```

### Why this is important:
  - Kibana talks to ES using **hostnames**
  - Avoid **DNS** issues
  - **Stable** connections

---

## Install Kibana from Elastic Repository

### Import Elastic GPG Key

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

#

### Define a Repository for Kibana

- Create repo file:

  ```bash
  vi /etc/yum.repos.d/elasticsearch.repo
  ```

- Add:

  ```ini
  [elasticsearch]
  name=Elastic repository
  baseurl=https://artifacts.elastic.co/packages/9.x/yum
  gpgcheck=1
  gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
  enabled=1
  autorefresh=1
  type=rpm-md
  ```

- Verify Repository:

  ```bash
  dnf repolist | grep elasticsearch
  ```

>[!Note]
For other **version** installation, change the **version name only** like **Elasticsearch 9.x**:

```bash
baseurl=https://artifacts.elastic.co/packages/<version-name>.x/yum
```

#

### Install Kibana from The Repository We Defined Earlier

- To Check all Available Kibana Version:

  ```bash
  dnf list --showduplicates kibana
  ```

- Now install required kibana:

  ```bash
  dnf install kibana-9.4.1
  ```

---


## Post-Installation Note (IMPORTANT)

⚠ Do NOT start Kibana yet.

Next steps (to be covered in the next SOP):

* kibana.yml configuration
* Elasticsearch connection setup
* TLS trust configuration
* Enrollment token or service account authentication
* Firewall configuration (port 5601)
* Kibana service start & verification