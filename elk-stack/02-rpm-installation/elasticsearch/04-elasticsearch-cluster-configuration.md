# Configure Elasticsearch Cluster (All Nodes)

Now Elasticsearch packages are **installed** on all 3 servers.

**At this stage:**

* Elasticsearch service exists
* Config files exist
* Security certificates are generated using Elasticsearch cert-util
* But cluster is NOT configured yet

**Now we will:**

1. Configure **node identity**
2. Configure cluster discovery
3. Assign node roles
4. Configure networking
5. Configure JVM heap
6. Configure systemd limits
7. Start cluster
8. Verify cluster formation

---

## Understand Elasticsearch Node Roles

Before configuration, understand the roles.

| VM  | Hostname  | IP (example)  | Role                   |
| --- | --------- | ------------- | ---------------------- |
| VM1 | es-node-1 | 192.168.10.11 | Master + Data + Ingest |
| VM2 | es-node-2 | 192.168.10.12 | Master + Data + Ingest |
| VM3 | es-node-3 | 192.168.10.13 | Master + Data + Ingest |

For small lab:

* All 3 nodes:

  * master
  * data
  * ingest

This is called:

* **Multi-purpose node architecture**

Good for:

* Labs
* Small production
* Learning

---

## Important Elasticsearch Paths

| Path                                   | Purpose               |
| -------------------------------------- | --------------------- |
| `/etc/elasticsearch/`                  | Main config directory |
| `/etc/elasticsearch/elasticsearch.yml` | Main configuration    |
| `/etc/elasticsearch/jvm.options.d/`    | JVM tuning            |
| `/var/lib/elasticsearch/`              | Data directory        |
| `/var/log/elasticsearch/`              | Log directory         |
| `/usr/share/elasticsearch/`            | ES binaries           |

---

## Configure ElasticSearch Node 1

Login to:

```bash
es-node-1
```

Edit config:

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

**Replace/Add Below Configuration**

```yaml
# ----------------------------
# Cluster Information
# ----------------------------

cluster.name: elk-prod-cluster

# ----------------------------
# Node Information
# ----------------------------

node.name: es-node-1

# Node Roles
node.roles:
  - master
  - data
  - ingest

# ----------------------------
# Paths
# ----------------------------

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# ----------------------------
# Network
# ----------------------------

network.host: 192.168.10.11

http.port: 9200

# ----------------------------
# Discovery
# ----------------------------

discovery.seed_hosts:
  - 192.168.10.11
  - 192.168.10.12
  - 192.168.10.13

cluster.initial_master_nodes:
  - es-node-1
  - es-node-2
  - es-node-3

# ----------------------------
# Security
# ----------------------------

xpack.security.enabled: true

# Transport SSL
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport/elastic-transport-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/transport/elastic-transport-certificates.p12

# HTTP SSL
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http/http.p12
```

Save and exit.

---

## Configure ElasticSearch Node 2

Login to:

```bash
es-node-2
```

Edit:

```bash
vim /etc/elasticsearch/elasticsearch.yml
```


**Replace/Add Below Configuration**

```yaml
cluster.name: elk-prod-cluster

node.name: es-node-2

node.roles:
  - master
  - data
  - ingest

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

network.host: 192.168.10.12

http.port: 9200

discovery.seed_hosts:
  - 192.168.10.11
  - 192.168.10.12
  - 192.168.10.13

cluster.initial_master_nodes:
  - es-node-1
  - es-node-2
  - es-node-3

xpack.security.enabled: true

# Transport SSL
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport/elastic-transport-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/transport/elastic-transport-certificates.p12

# HTTP SSL
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http/http.p12
```

---

# 4.5 Configure ElasticSearch Node 3

Login to:

```bash
es-node-3
```

Edit:

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

**Replace/Add Below Configuration**

```yaml
cluster.name: elk-prod-cluster

node.name: es-node-3

node.roles:
  - master
  - data
  - ingest

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

network.host: 192.168.10.13

http.port: 9200

discovery.seed_hosts:
  - 192.168.10.11
  - 192.168.10.12
  - 192.168.10.13

cluster.initial_master_nodes:
  - es-node-1
  - es-node-2
  - es-node-3

xpack.security.enabled: true

# Transport SSL
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport/elastic-transport-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/transport/elastic-transport-certificates.p12

# HTTP SSL
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http/http.p12
```

---

## Bootstrap Checks

When Elasticsearch binds to a **non-localhost** network interface using:
```
network.host
```
- Elasticsearch enters **production mode**.
- In production mode, Elasticsearch enforces **bootstrap checks** to ensure the system is **properly configured** for stability and performance.

Common bootstrap checks include:

- [vm.max_map_count](01-system-preparation.md#25-increase-virtual-memory)
- [swap disabled](01-system-preparation.md#)
- JVM heap settings
- file descriptor limits
- memory locking

If these checks fail, Elasticsearch will refuse to start.

>We configured some setting during system tuning and now we will configure the rest.


## Configure JVM Heap Size

### JVM Heap Rules

Default JVM heap is often not ideal.

Rule:

* Heap = 50% of RAM
* Never more than 31 GB

Your nodes:
```text
RAM = 4 GB
```
Recommended:

```text
2 GB heap
```

#

### Edit JVM Options

On ALL nodes:

```bash
vim /etc/elasticsearch/jvm.options.d/heap.options
```

Add below line:

```text
-Xms2g
-Xmx2g
```

Meaning:

| Option | Purpose      |
| ------ | ------------ |
| Xms    | Initial heap |
| Xmx    | Maximum heap |

IMPORTANT:

* Both values MUST be same

---

## Configure Systemd File Desciptor Limits

### Earlier we configured:

```bash
/etc/security/limits.d/elasticsearch.conf
```

But systemd ignores those [File Discriptor](01-system-preparation.md#26-file-descriptor-limits) limits as it is installed by package manager.

#

### Now configure real service limits.

Create Override Directory On ALL nodes:

```bash
mkdir -p /etc/systemd/system/elasticsearch.service.d
```

Create Override File

```bash
vim /etc/systemd/system/elasticsearch.service.d/override.conf
```

Add:

```ini
[Service]
LimitNOFILE=65535
LimitNPROC=4096
LimitMEMLOCK=infinity
```
Note: `LimitMEMLOCK=infinity` - Allows Elasticsearch to lock JVM memory and prevent swapping.

Reload systemd

```bash
systemctl daemon-reload
```

---

## Enable Memory Locking

Edit:

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

Add:

```yaml
bootstrap.memory_lock: true
```

Purpose:

```
Prevent JVM memory swapping
```

---

## Verify Elasticsearch User Permissions

Check ownership:

```bash
ls -ld /var/lib/elasticsearch
ls -ld /var/log/elasticsearch
```

Expected:

```text
elasticsearch elasticsearch
```

Fix if needed:

```bash
chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
chown -R elasticsearch:elasticsearch /var/log/elasticsearch
```

---

## Add Secure Passwords To Elasticsearch Keystore

Run the following commands on **ALL Elasticsearch nodes**.

#

**Add HTTP SSL Keystore Password**

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
```

You will be prompted to enter the HTTP certificate password.


**Add Transport SSL Keystore Password**

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
```

#

### Verify Stored Secure Settings

**List configured secure settings:**

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore list
```

**Expected output example:**

```text
xpack.security.http.ssl.keystore.secure_password
xpack.security.transport.ssl.keystore.secure_password
```

>[!NOTE]
Only setting names are displayed.\
Secret values are never shown.

#

### Varify Secure Elasticsearch Keystore Permissions

**Verify permissions:**

```bash
ls -l /etc/elasticsearch/elasticsearch.keystore
```

**Recommended:**

```text
-rw------- 1 root elasticsearch
```

**Fix permissions if required:**

```bash
chmod 600 /etc/elasticsearch/elasticsearch.keystore
chown root:elasticsearch /etc/elasticsearch/elasticsearch.keystore
```

>[!NOTE]
Although the file permission is 600, Elasticsearch can still access the keystore through its privileged startup and internal secure settings handling mechanism.

---

## Start Elasticsearch Cluster

### Enable ES Service

```bash
systemctl daemon-reload
systemctl enable elasticsearch
```

#

### Start ES Services - Order Matters First Time

Start **one by one**, not all together.

**On Elasticsearch node-1**

```bash
systemctl start elasticsearch
```

>Wait 1–2 minutes.

**On Elasticsearch node-2**

```bash
systemctl start elasticsearch
```

>Wait.

**On Elasticsearch node-3**

```bash
systemctl start elasticsearch
```

#

### Check Service Status

```bash
systemctl status elasticsearch
```

Must be:

```text
Active: active (running)
```

---

## Monitor Elasticsearch Logs

VERY IMPORTANT during first startup.

Run:

```bash
journalctl -u elasticsearch -f
```

OR:

```bash
tail -f /var/log/elasticsearch/elasticsearch.log
```

---

## Verify Cluster Status

### Check Cluster Health

From any node:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cluster/health?pretty
```

Expected:

```json
{
  "cluster_name" : "elk-prod-cluster",
  "status" : "green",
  "number_of_nodes" : 3
}
```

Cluster health status:

- green:\
  all primary and replica shards allocated

- yellow:\
  primary shards allocated but some replicas missing

- red:\
  some primary shards unavailable

---

## Common Beginner Mistakes

- Forgetting swapoff
- Wrong IP in `network.host`
- Firewall not opened
- Heap size too large
- Starting all nodes at once first time
- Hostname not resolvable

---

## Important Production Warning

Large production clusters usually separate:
- dedicated master nodes
- dedicated data nodes
- ingest nodes
- coordinating nodes

---

## Other Installation Guides:
- Previous: [Elasticsearch TLS Certificate Generation](03-elasticsearch-tls-certificate-generation.md)
- Next: 


