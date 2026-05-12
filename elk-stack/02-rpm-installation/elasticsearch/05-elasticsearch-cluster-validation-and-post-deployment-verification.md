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

Run on ALL nodes:

```bash
systemctl status elasticsearch
```

Expected:

```text
active (running)
```

---

## 3. Validate Listening Ports

Check listening ports:

```bash
ss -tulnp | grep -E '9200|9300'
```

Expected:

* `9200` → HTTPS API
* `9300` → Transport layer

This validates proper network binding and listening services.

---

## 4. Validate TLS Connectivity

Very important.

### Check HTTPS Connectivity

```bash
curl -k -u elastic https://192.168.10.11:9200
```

Expected:

* valid JSON response
* cluster information

---

### Validate TLS Certificate

```bash
openssl s_client -connect 192.168.10.11:9200
```

This validates:

* TLS handshake
* certificate presentation
* CA chain validation

This is an excellent operational troubleshooting step.

---

## 5. Retrieve or Reset Elastic Password

### Retrieve Auto-Generated Password

```bash
grep "generated password" /var/log/elasticsearch/elasticsearch.log
```

> [!NOTE]
> Elasticsearch 8.x and later versions automatically generate a password during the initial startup.

---

### Reset Password (If Initial Password Is Lost)

```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

> [!NOTE]
> You will be prompted to set a custom password.

---

### Verify Authentication

```bash
curl -k -u elastic https://192.168.10.11:9200
```

Expected:

* valid JSON response

---

## 6. Validate Cluster Health

Cluster health validation confirms:

* shard allocation
* node membership
* cluster coordination

### Check Cluster Health

Run from any node:

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

### Understanding Cluster State Colors

| Color  | Meaning                    |
| ------ | -------------------------- |
| green  | All shards are healthy     |
| yellow | Replica shards are missing |
| red    | Primary shards are missing |

>[!TIP] 
For a 3-node cluster, aim for `GREEN` status.

---

## 7. Validate Node Status

### Validate Node Membership

Run from any node:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cat/nodes?v
```

Validate:

* all expected nodes joined
* node roles are correct
* one elected master exists

---

### Validate Elected Master Node

Run from any node:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cat/master?v
```

This validates:

* elected master node

---

## 8. Validate Shard Allocation

Very important operational check.

```bash
curl -k -u elastic https://192.168.10.11:9200/_cat/shards?v
```
Or:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cat/allocation?v
```

This validates:

* _cat/shards shows shard states
* _cat/allocation shows disk allocation balance

This is an excellent real-world operational check.

---

## 9. Validate JVM Heap

Very important.

```bash
curl -k -u elastic https://192.168.10.11:9200/_nodes/jvm?pretty
```

Verify:

* `Xms`
* `Xmx`
* heap size

This confirms that JVM tuning has been applied correctly.

---

## 10. Validate Memory Locking

Very important.

```bash
curl -k -u elastic https://192.168.10.11:9200/_nodes?filter_path=**.mlockall&pretty
```

Expected:

```json
"mlockall" : true
```

This confirms:

* swap prevention
* memory locking is active

---

## 11. Validate File Descriptor Limits

Very good operational validation.

Get the Elasticsearch PID:

```bash
pgrep -f org.elasticsearch.bootstrap.Elasticsearch
```

Then run:

```bash
cat /proc/<PID>/limits | grep "Max open files"
```

Expected:

```text
65535
```

---

## 12. Remove Bootstrap Setting (CRITICAL)

> [!IMPORTANT]
> `cluster.initial_master_nodes` is required only during the initial cluster bootstrap process.
>
> Leaving it permanently configured may cause future cluster formation problems.

After the cluster becomes healthy:

Edit the configuration file on ALL nodes:

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

Remove the following section:

```yaml
cluster.initial_master_nodes:
  - es-node-1
  - es-node-2
  - es-node-3
```

> [!IMPORTANT]
> Do NOT restart all nodes simultaneously.

---

## 13. Safe Rolling Restart

Perform a safe rolling restart of the cluster nodes.

### Restart Node 1

```bash
systemctl restart elasticsearch
```

---

### After Restarting Each Node

Validate cluster health:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cluster/health?pretty
```

Ensure:

* cluster status returns `GREEN` or expected `YELLOW`
* restarted node successfully rejoins the cluster

Then continue with:

* Node2
* Node3

---

## 14. Validate Cluster After Restart

After the rolling restart, validate again:

* Cluster Health
* Node Membership
* Master Election
* Shard Recovery

---

## 15. Validate Elasticsearch Logs

Very important.

```bash
journalctl -u elasticsearch -n 50
```

Look for:

* bootstrap errors
* TLS issues
* cluster join failures

This is a very useful troubleshooting step.

---

## 16. Validate Disk Space

Very important operational check.

```bash
df -h
```

>[!IMPORTANT]
Elasticsearch may stop shard allocation when disk usage becomes too high


---


## 18. Optional but VERY Professional — Test Index Creation

This validates:

* indexing functionality
* shard creation
* cluster write capability

### Create Test Index

```bash
curl -k -u elastic -X PUT https://192.168.10.11:9200/test-index
```

---

### Insert Test Document

```bash
curl -k -u elastic -X POST https://192.168.10.11:9200/test-index/_doc \
-H "Content-Type: application/json" \
-d '{"name":"elastic-test"}'
```

---

### Search Test Data

```bash
curl -k -u elastic https://192.168.10.11:9200/test-index/_search?pretty
```

---

### Cleanup Test Index

```bash
curl -k -u elastic -X DELETE https://192.168.10.11:9200/test-index
```

> This is an extremely valuable operational validation step.

---

## 19. Validate Security Features

> [!NOTE]
> Security features are sometimes accidentally disabled during configuration changes.

### Validate Security Configuration

```bash
curl -k -u elastic https://192.168.10.11:9200/_xpack?pretty
```

OR

```bash
curl -k -u elastic https://192.168.10.11:9200/_security/_authenticate?pretty
```

This confirms:

* authentication is working
* security features are enabled

---

## 20. Validate Elasticsearch Keystore

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore list
```

Verify required secure settings exist.

Example:

* xpack.security.http.ssl.keystore.secure_password
* xpack.security.transport.ssl.keystore.secure_password

Validate keystore file permissions::

```bash
ls -l /etc/elasticsearch/elasticsearch.keystore
```

Expected:

```text
-rw------- elasticsearch elasticsearch
```


---


## 21. Validate Snapshot Repository

Example:

```bash
curl -k -u elastic https://192.168.10.11:9200/_snapshot?pretty
```

Validate:

* repository configured
* repository is accessible and writable

---

## 22. Node Failure Simulation

### Example

Stop one node:

```bash
systemctl stop elasticsearch
```

Validate:

* cluster remains operational
* quorum is maintained
* shards are reallocated properly

Then restart the node.

This validates:

* high availability (HA)
* real production resiliency

---

## 23. Final Readiness Checklist

| Validation                | Status |
| ------------------------- | ------ |
| Cluster health is green   | ✅      |
| All 3 nodes joined        | ✅      |
| TLS is working            | ✅      |
| Heap is configured        | ✅      |
| Memory locking enabled    | ✅      |
| Bootstrap setting removed | ✅      |
| Test indexing successful  | ✅      |

---
