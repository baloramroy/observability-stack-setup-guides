# Elasticsearch Cluster Validation & Post-Deployment Verification

## 1. Purpose of This SOP

This SOP validates:
- Elasticsearch cluster health
- node communication
- TLS functionality
- security configuration
- JVM configuration
- shard allocation
- bootstrap cleanup
- cluster readiness

Excellent operational framing.

---

## 2. Validate Elasticsearch Service Status

On ALL nodes:

```bash
systemctl status elasticsearch
```

Expected:

```text
active (running)
```

---

## 3. Validate Listening Ports

Check:

```bash
ss -tulnp | grep -E '9200|9300'
```

Expected:

* 9200 → HTTPS API
* 9300 → transport layer

This validates network binding.

---

## 4. Validate TLS Connectivity

Very important.

#

### Check HTTPS Connectivity

```bash
curl -k -u elastic https://192.168.10.11:9200
```

Expected:

* JSON response
* cluster info

#

### Validate TLS Certificate

Very valuable addition:

```bash
openssl s_client -connect 192.168.10.11:9200
```

Validates:

* TLS handshake
* certificate presentation
* CA chain

Excellent operational troubleshooting step.

---

## 5. Retrieve or Reset Elastic Password

**Retrieve Auto-Generated Password**

```bash
grep "generated password" /var/log/elasticsearch/elasticsearch.log
```
>[!NOTE] 
>Elasticsearch 8.x or later version auto-generates password.

**If initial password was lost, reset it:**

```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

>[!NOTE] 
>You will be prompted for custom password

**Then:**

```bash
curl -k -u elastic https://192.168.10.11:9200
```

>You will get a valid json response

---

## 6. Validate Cluster Health

Cluster health validates:
- shard allocation
- node membership
- cluster coordination

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

Understanding Cluster State Colors

| Color  | Meaning               |
| ------ | --------------------- |
| green  | All shards healthy    |
| yellow | Replica missing       |
| red    | Primary shard missing |

>For 3-node cluster: `Aim for GREEN`


---

## 7. Validate Node Status

### Validate Node Membership:

Run from Any Node:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cat/nodes?v
```

You should see:

```text
es-node-1
es-node-2
es-node-3
```

#

### Validate Master Node:

Run from Any Node:
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

Expected:

* STARTED state

This validates:

* shard placement
* replication

Excellent real-world check.

---

## 9. Validate JVM Heap

Very important.

```bash
curl -k -u elastic https://192.168.10.11:9200/_nodes/jvm?pretty
```

Check:

* Xms
* Xmx
* heap size

Confirms JVM tuning applied.

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

Confirms:

* swap prevention
* memory locking active

---

## 11. Validate File Descriptor Limits

Very good operational validation.

Get ES PID:

```bash
pidof java
```

Then:

```bash
cat /proc/<PID>/limits | grep "Max open files"
```

Expected:

```text
65535
```

---


## 14. Remove Bootstrap Setting (CRITICAL)

>[!IMPORTANT]
`cluster.initial_master_nodes` is only required during **first cluster bootstrap**.\
Leaving it **permanently **configured may cause **future cluster formation problems**.


**After cluster is healthy:**

Edit ALL nodes:

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

Remove this section:

```yaml
cluster.initial_master_nodes:
  - es-node-1
  - es-node-2
  - es-node-3
```
>[!IMPORTANT] 
DO NOT restart all nodes simultaneously.

---

## 15. Safe Rolling Restart

**Restart safely:**

Node1:

```bash
systemctl restart elasticsearch
```

#

**After restarting each node:**

Validate:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cluster/health?pretty
```

Ensure:

- cluster returns **GREEN** or expected **YELLOW**
- restarted node **rejoins** cluster

THEN continue next node:

* Node2
* Node3

---

# 16. Validate Cluster After Restart

After Restart, validate again:
* [Cluster Health](#6-validate-cluster-health)
* [Node count](#validate-node-membership)
* [master election](#validate-master-node)
* [Shard recovery](#9-validate-shard-allocation)

---

## 17. Validate Elasticsearch Logs

Very important.

```bash
journalctl -u elasticsearch -n 50
```

Look for:

* bootstrap errors
* TLS issues
* cluster join failures

Very useful troubleshooting step.

---

## 18. Validate Disk Space

Very important operational check.

```bash
df -h
```

Especially important because:

* Elasticsearch stops allocation when disks fill

Good operational awareness.

---

## 19. Validate Time Synchronization

Good final validation.

```bash
timedatectl status
```

Confirms:

* NTP
* clock sync

Important for:

* TLS validity
* log analysis

---

## 20. Optional But VERY Professional — Test Index Creation

It validates:

* indexing
* shard creation
* cluster write capability

Create test index:

```bash
curl -k -u elastic -X PUT https://192.168.10.11:9200/test-index
```

Insert document:

```bash
curl -k -u elastic -X POST https://192.168.10.11:9200/test-index/_doc \
-H "Content-Type: application/json" \
-d '{"name":"elastic-test"}'
```

Search:

```bash
curl -k -u elastic https://192.168.10.11:9200/test-index/_search?pretty
```

Cleanup Test Index after successful test:

```bash
curl -k -u elastic -X DELETE https://192.168.10.11:9200/test-index
```
>This is EXTREMELY valuable operational validation.


---

## Validate Security Features Enabled

>[!NOTE]
Sometimes admins accidentally disable security.

Validate Security Configuration

```bash
curl -k -u elastic https://192.168.10.11:9200/_xpack?pretty
```

OR:

```bash
curl -k -u elastic https://192.168.10.11:9200/_security/_authenticate?pretty
```

Confirms:

- authentication working
- security enabled


---

## Node Failure Simulation

**Example:**

Stop one node:

```bash
systemctl stop elasticsearch
```

Validate:

- cluster remains operational
- quorum maintained
- shards reallocate

Then restart node.

This becomes:

- HA validation
- real production readiness test


---

## 21. Final Readiness Checklist

Example:

| Validation                | Status  |
| ------------------------- | ------  |
| Cluster green             | ✅      |
| 3 nodes joined            | ✅      |
| TLS working               | ✅      |
| Heap configured           | ✅      |
| Memory locking enabled    | ✅      |
| Bootstrap setting removed | ✅      |
| Test indexing successful  | ✅      |

---
