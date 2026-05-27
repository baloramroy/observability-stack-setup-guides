# Elasticsearch Cluster Validation & Post-Deployment Verification

## 1. Purpose of This SOP

This SOP validates:

* Elasticsearch cluster health
* node communication
* TLS functionality
* security configuration
* JVM configuration
* shard allocation
* bootstrap cleanup
* overall cluster readiness

---

## 2. Validate Elasticsearch Service Status

- Run on ALL nodes:

  ```bash
  systemctl status elasticsearch
  ```

- Expected:

  ```text
  active (running)
  ```

---

## 3. Validate Listening Ports

- Check listening ports:

  ```bash
  ss -tulnp | grep -E '9200|9300'
  ```

- Expected:

  * `9200` → HTTPS API
  * `9300` → Transport layer

  >This validates proper network binding and listening services.

---

## 4. Retrieve or Reset Elastic Password

- Retrieve Auto-Generated Password

  ```bash
  grep "generated password" /var/log/elasticsearch/elasticsearch.log
  ```

  > Elasticsearch 8.x and later versions automatically generate a password during the initial startup.


- Reset Password (If Initial Password Is Lost)

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
  ```

  > You will be prompted to set a custom password.


- Verify Authentication

  ```bash
  curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200
  ```

- Expected:

  * valid JSON response


---

## 5. Validate TLS Connectivity

Very important.

### Check HTTPS Connectivity

- Validate HTTP TLS handshake:

  ```bash
  curl -k -u elastic:PASS https://192.168.0.124:9200
  ```

- Expected:

  * valid JSON response
  * cluster information

#

### Validate Transport Layer Security


- Validate transport TLS handshake:

  ```bash
  openssl s_client -connect 192.168.10.11:9300
  ```

- This validates:

  * inter-node encrypted communication
  * transport certificate trust
  * successful TLS-based cluster coordination


---

## 6. Validate Cluster Health

- Run from any node:

  ```bash
  curl -k -u elastic:PASS https://192.168.0.124:9200/_cluster/health?pretty
  ```

- Expected:

  ```json
  {
    "cluster_name" : "elk-prod-cluster",
    "status" : "green",
    "number_of_nodes" : 3
  }
  ```

- Cluster Health Status:

  - Green 🟢:\
    All primary and replica shards allocated

  - Yellow 🟡:\
    Primary shards allocated but some replicas missing

  - Red 🔴:\
    Some primary shards unavailable

> [!TIP]
> For a 3-node cluster, aim for `GREEN` status.

---

## 7. Validate Elasticsearch Logs

- Very important.

  ```bash
  journalctl -u elasticsearch -n 50
  ```

- Look for:

  * bootstrap errors
  * TLS issues
  * cluster join failures

>This is a very useful troubleshooting step.

---

## 8. Validate Node Status & Master

### Validate Node Membership

- Run from any node:

  ```bash
  curl -k -u elastic:PASS https://192.168.0.124:9200/_cat/nodes?v
  ```

- Output:

  ```
  ip             heap.percent ram.percent  cpu  node.role  master  name
  192.168.0.124            19          96    0  dim        *       es-node-1
  192.168.0.125            71          95    1  dim        -       es-node-2
  192.168.0.126            56          99    1  dim        -       es-node-3
  ```

- Role Decoding

  - `m`  → master node
  - `d`  → data node
  - `i`  → ingest node
  - `*`  → current elected master

- Validate:

  * all expected nodes joined
  * node roles are correct
  * one elected master exists

---

### Validate Elected Master Node

- Run from any node:

  ```bash
  curl -k -u elastic:PASS https://192.168.0.124:9200/_cat/master?v
  ```

- Output:

  ```
  id                      host           ip             node
  XP6aYZieRR2wgcJ5CpZocQ  192.168.0.124  192.168.0.124  es-node-1
  ```

- This validates:

  * elected master node

---

## 9. Validate Shard Allocation

Shard allocation validation ensures:

- Data is properly distributed
- Replication is working
- No node is overloaded or unhealthy

#

### Check Shard Distribution

**Command:**

```bash
curl -k -u elastic https://192.168.0.124:9200/_cat/shards?v
```

**Example Output:**

```
index                                 shard prirep state   docs  store dataset ip            node
.ds-.logs-elasticsearch.deprecation.. 0     p      STARTED    2   12kb    12kb 192.168.0.126 es-node-3
.ds-.logs-elasticsearch.deprecation.. 0     r      STARTED    2   12kb    12kb 192.168.0.125 es-node-2
.security-7                           0     p      STARTED   30 45.3kb  45.3kb 192.168.0.124 es-node-1
.security-7                           0     r      STARTED   30 45.3kb  45.3kb 192.168.0.125 es-node-2
.ds-ilm-history-7-2026.05.27-000001   0     p      STARTED    3   11kb    11kb 192.168.0.124 es-node-1
.ds-ilm-history-7-2026.05.27-000001   0     r      STARTED    3 11.1kb  11.1kb 192.168.0.126 es-node-3
```

**Interpretation**

1. Shard Types

   * `p` → Primary shard
   * `r` → Replica shard


2. Health State

   * `STARTED` → Shard is active and fully operational
   * Any other state (e.g. INITIALIZING, UNASSIGNED) → potential issue


3. Key Observation

    Example:

    ```
    .security-7
    p → es-node-1
    r → es-node-2
    ```

    Meaning:

    * Data is replicated across nodes
    * High availability is working correctly


4. System Indices

    Common system indices:

    * `.security-7` → Authentication and security data
    * `.ds-ilm-history-*` → Index lifecycle management history
    * `.logs-*` → Logging data streams


#

### Check Cluster Allocation & Disk Usage

**Command**

```bash
curl -k -u elastic:PASS https://192.168.0.124:9200/_cat/allocation?v
```

**Output:**

```
shards shards.undesired disk.indices disk.used disk.avail disk.total host          node      node.role
     2                0       56.4kb     4.2gb     12.6gb     16.9gb 192.168.0.124 es-node-1 dim
     2                0       57.3kb     4.1gb     12.8gb     16.9gb 192.168.0.125 es-node-2 dim
     2                0       23.1kb     4.1gb     12.8gb     16.9gb 192.168.0.126 es-node-3 dim
```

**Interpretation**

1. Shard Distribution

   * Each node holds a small number of shards
   * No imbalance observed


2. Disk Usage

   * `disk.used` → actual storage used
   * `disk.avail` → free space
   * `disk.total` → total capacity


3. Key Health Check

   - ✔ No node is overloaded
   - ✔ No disk pressure detected
   - ✔ Shards are evenly distributed

---

## 10. Validate JVM Heap

- Very important.

  ```bash
  curl -k -u elastic https://192.168.0.124:9200/_nodes/jvm?pretty
  ```

- Verify:

  * `Xms`
  * `Xmx`
  * heap size

  This confirms that JVM tuning has been applied correctly.

---

## 11. Validate Memory Locking

- Very important.

  ```bash
  curl -k -u elastic https://192.168.0.124:9200/_nodes?filter_path=**.mlockall
  ```

- Expected:

  ```json
  "mlockall" : true
  ```

  This confirms:

  * swap prevention
  * memory locking is active

---

## 12. Validate File Descriptor Limits

- Get the Elasticsearch PID:

  ```bash
  pgrep -f org.elasticsearch.bootstrap.Elasticsearch
  ```

- Then run:

  ```bash
  cat /proc/<PID>/limits | grep "Max open files"
  ```

- Expected:

  ```text
  65535
  ```

---

## 13. Remove Bootstrap Setting (CRITICAL)

> [!IMPORTANT]
> `cluster.initial_master_nodes` is required only during the initial cluster bootstrap process.\
> Leaving it permanently configured may cause future cluster formation problems.

After the cluster becomes **healthy**:

- Edit the configuration file on ALL nodes:

  ```bash
  vim /etc/elasticsearch/elasticsearch.yml
  ```

- Remove the following section:

  ```yaml
  cluster.initial_master_nodes:
    - es-node-1
    - es-node-2
    - es-node-3
  ```

> [!IMPORTANT]
Do NOT restart all nodes simultaneously.\
Perform a safe rolling restart of the cluster nodes.

- Restart Node 1

  ```bash
  systemctl restart elasticsearch
  ```

- After Restarting Validate cluster health:

  ```bash
  curl -k -u elastic:PASS https://192.168.0.124:9200/_cluster/health?pretty
  ```

  Ensure:

  * cluster status returns `GREEN` or expected `YELLOW`
  * restarted node successfully rejoins the cluster
  * restarted node appears in `_cat/nodes`

- Then continue with:

  * Node2
  * Node3

---

## 15. Validate Cluster After Restart

After the rolling restart, validate again:

* Cluster Health
* Node Membership
* Master Election
* Shard Recovery

---

## 16. Validate Disk Space

- Very important operational check.

  ```bash
  df -h
  ```

> [!IMPORTANT]
> Elasticsearch may stop shard allocation when disk usage becomes too high.

---

## 17. Optional but VERY Professional — Test Index Creation

**This validates:**

* indexing functionality
* shard creation
* cluster write capability

#

**Test Index Creation:**

- Create Test Index

  ```bash
  curl -k -u elastic -X PUT https://192.168.0.124:9200/test-index
  ```


- Insert Test Document

  ```bash
  curl -k -u elastic -X POST https://192.168.0.124:9200/test-index/_doc \
  -H "Content-Type: application/json" \
  -d '{"name":"elastic-test"}'
  ```


- Search Test Data

  ```bash
  curl -k -u elastic https://192.168.0.124:9200/test-index/_search?pretty
  ```


- Cleanup Test Index

  ```bash
  curl -k -u elastic -X DELETE https://192.168.0.124:9200/test-index
  ```

> This is an extremely valuable operational validation step.

---

## 18. Validate Security & Features

> [!NOTE]
> Security features are sometimes accidentally disabled during configuration changes.

### Validate Security Authentication

- Run below command
  ```bash
  curl -k -u elastic https://192.168.0.124:9200/_security/_authenticate?pretty
  ```

- Output:

  ```json
  {
  "username" : "elastic",
    "roles" : [
      "superuser"
    ],
    "full_name" : null,
    "email" : null,
    "metadata" : {
      "_reserved" : true
    },
  }
  ```

- This confirms:

  * authentication is working
  * security features are enabled

#

### Validate License Status 

- Feature and license status report

  ```bash
  curl -k -u elastic https://192.168.0.124:9200/_xpack?pretty
  ```

- Output:

  ```json
  {
    "build" : {
      "hash" : "3c7c6027c5769d860d87448e2749f4c550a239da",
      "date" : "2026-05-08T10:08:29.383338563Z"
    },
    "license" : {
      "uid" : "5cc4ced7-2680-47c3-9a7a-2fcc9302fcda",
      "type" : "basic",
      "mode" : "basic",
      "status" : "active"
    },
  }
  ```
---

## 19. Validate Elasticsearch Keystore

- Verify required secure settings exist

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-keystore list
  ```

  Example:

  * `xpack.security.http.ssl.keystore.secure_password`
  * `xpack.security.transport.ssl.keystore.secure_password`
  * `xpack.security.transport.ssl.truststore.secure_password`


- Validate keystore file permissions:

  ```bash
  ls -l /etc/elasticsearch/elasticsearch.keystore
  ```

  Expected:

  ```text
  -rw------- elasticsearch elasticsearch
  ```

---

## 20. Validate Snapshot Repository

- Example:

  ```bash
  curl -k -u elastic https://192.168.0.124:9200/_snapshot?pretty
  ```

- Validate:

  * repository configured
  * repository is accessible and writable

---

## 21. Node Failure Simulation

- Stop one node:

  ```bash
  systemctl stop elasticsearch
  ```

- Validate:

  * cluster remains operational
  * quorum is maintained
  * shards are reallocated properly

  >Then restart the node.

- This validates:

  * high availability (HA)
  * real production resiliency

---

## 22. Final Readiness Checklist

| Validation                    | Status |
| ----------------------------- | ------ |
| Cluster health is green       | ✅      |
| All 3 nodes joined            | ✅      |
| TLS is working                | ✅      |
| Heap is configured            | ✅      |
| Memory locking enabled        | ✅      |
| Bootstrap setting removed     | ✅      |
| Test indexing successful      | ✅      |
| Keystore validated            | ✅      |
| Snapshot repository validated | ✅      |

---
