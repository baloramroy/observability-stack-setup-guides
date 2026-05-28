# Elasticsearch Operational Alerting SOP

## 1. Cluster Health Alerts

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cluster/health?pretty
```

#

### Alert Conditions

| Condition                     | Severity         | Alert                     |
| ----------------------------- | ---------------- | ------------------------- |
| status = green                | OK               | Healthy                   |
| status = yellow               | Warning          | Replica allocation issue  |
| status = red                  | Critical         | Primary shard unavailable |
| unassigned_shards > 0         | Warning          | Allocation problem        |
| delayed_unassigned_shards > 0 | Warning          | Delayed allocation        |
| active_shards_percent < 100   | Warning/Critical | Missing shards            |

#

### Investigate

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cluster/allocation/explain?pretty
```

Check:

* disk watermark
* node failure
* allocation filtering
* corrupted shard

---

# 2. Node Availability Alerts

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cat/nodes?v
```

#

### Alert Conditions

| Condition                | Severity |
| ------------------------ | -------- |
| expected node missing    | Critical |
| node disconnected        | Critical |
| master node unavailable  | Critical |
| frequent node join/leave | Warning  |

#

### Investigate

Check:

* server down
* JVM crash
* OOM
* network issue
* split brain
* transport TLS issue

Commands:

```bash
systemctl status elasticsearch
journalctl -u elasticsearch -xe
```

---

# 3. Master Election Alerts

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cat/master?v
```

#

### Alert Conditions

| Condition                 | Severity |
| ------------------------- | -------- |
| no master elected         | Critical |
| master changes frequently | Warning  |
| cluster state unstable    | Warning  |


#

### Investigate

Check:

* quorum issue
* network partition
* discovery problem
* overloaded master

Logs:

```bash
journalctl -u elasticsearch | grep master
```

---

## 4. Unassigned Shard Alerts

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cat/shards?v
```

#

### Threshold

| Metric                            | Threshold |
| --------------------------------- | --------- |
| unassigned replica > 0 for 10 min | Warning   |
| unassigned primary > 0            | Critical  |
| initializing shard > 30 min       | Warning   |

#

### Investigate

Check:

* disk full
* node loss
* allocation disabled
* shard corruption

Command:

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cluster/allocation/explain?pretty
```

---

## 5. JVM / Heap Alerts

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_nodes/stats/jvm?pretty
```

#

### Important Metrics

| Metric                        | Recommended |
| ----------------              | ----------- |
| heap usage                    | < 75%       |
| young GC                      | normal      |
| JVM pause                     | low         |
| old GC time > 5 sec           | Warning     |
| GC frequency spike            | Warning     |
| heap after GC remains > 75%   | Critical    |

#

### Alert Conditions

| Condition    | Severity |
| ------------ | -------- |
| heap > 75%   | Warning  |
| heap > 85%   | Critical |
| heap > 95%   | Critical |
| OOM detected | Critical |
| frequent GC  | Warning  |

#

### Investigate

Check:

* oversized shard
* insufficient heap
* memory leak
* heavy aggregation query
* indexing spikes

Logs:

```bash
journalctl -u elasticsearch | grep -Ei "gc|heap|oom"
```

---

## 6. Disk Usage Alerts

### Check

```bash
df -h
```

AND

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cat/allocation?v
```

#

### Elasticsearch Watermarks

| Watermark   | Default |
| ----------- | ------- |
| low         | 85%     |
| high        | 90%     |
| flood_stage | 95%     |

#

### Alert Conditions

| Condition                 | Severity |
| ------------------------- | -------- |
| disk > 80%                | Warning  |
| disk > 90%                | Critical |
| flood stage reached       | Critical |
| read-only index triggered | Critical |


#

### Investigate

Check:

* large indices
* snapshot issue
* log growth
* old indices retention

Commands:

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cat/indices?v&s=store.size:desc
```

---

## 7. Elasticsearch Log Alerts

### Check

```bash
journalctl -u elasticsearch -xe
```

#

### Critical Log Patterns

| Pattern                        | Severity |
| ------------------------------ | -------- |
| OutOfMemoryError               | Critical |
| master not discovered          | Critical |
| shard failed                   | Warning  |
| corrupted index                | Critical |
| flood stage watermark exceeded | Critical |
| GC overhead limit exceeded     | Critical |

#

### Recommended Monitoring

Alert if logs contain:

```bash
OutOfMemoryError
master not discovered
corrupt
flood stage
GC overhead
unable to allocate
```

---

## 8. Performance Alerts

### Check

```bash
curl -k -u elastic:PASS https://192.168.0.124:9200/_cat/thread_pool?v
```

AND

```bash
curl -k -u elastic:PASS https://192.168.0.124:9200/_nodes/hot_threads?pretty
```



### Alert Conditions

| Condition                 | Severity |
| ------------------------- | -------- |
| thread queue buildup      | Warning  |
| rejected tasks increasing | Critical |
| CPU > 85%                 | Warning  |
| CPU > 95%                 | Critical |
| hot threads persistent    | Warning  |



### Threshold

| Metric                                    | Threshold |
| --------------------                      | --------- |
| CPU > 85% for 5 min                       | Warning   |
| CPU > 95% for 5 min                       | Critical  |
| rejected tasks continuously increasing    | Warning   |
| search queue growing                      | Warning   |



### Investigate

Check:

* expensive queries
* excessive indexing
* shard imbalance
* insufficient CPU
* runaway aggregation

---

## 9. File Descriptor Alerts

Elasticsearch heavily depends on file descriptors.

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_nodes/stats/process?pretty
```

#

### Alert Conditions

| Condition          | Severity |
| ------------------ | -------- |
| FD usage > 80%     | Warning  |
| FD exhaustion risk | Critical |

---

## 10. Cluster Pending Tasks Alerts

### Check

```bash
curl -k -u elastic:8wCPuSah*ZPTDs_GE6MY https://192.168.0.124:9200/_cluster/pending_tasks?pretty
```

### Alert Conditions

| Condition                | Severity |
| ------------------------ | -------- |
| pending tasks increasing | Warning  |
| long queue               | Critical |

### Threshold

| Metric             | Threshold |
| ------------------ | --------- |
| queue > 10         | Warning   |
| queue > 50         | Critical  |
| task wait > 30 sec | Critical  |

### Investigate

Check:

* master overload
* mapping explosion
* shard allocation storm
* slow cluster state update

---

## Recommended Production Alerts Summary

| Category          | Warning  | Critical  |
| ----------------- | -------- | --------- |
| Cluster Health    | Yellow   | Red       |
| Heap Usage        | >75%     | >85%      |
| Disk Usage        | >80%     | >90%      |
| Flood Stage       | —        | >95%      |
| CPU               | >85%     | >95%      |
| Node Availability | —        | Node down |
| Unassigned Shards | Replica  | Primary   |
| Cert Expiry       | <30d     | <7d       |
| Auth Failures     | >10/min  | >50/min   |
| Master Changes    | Frequent | No master |

---

# Best Production Practice

We should monitor Elasticsearch from 3 layers:

| Layer                 | Tool                         |
| --------------------- | ---------------------------- |
| OS Metrics            | Node Exporter / Zabbix Agent |
| Elasticsearch Metrics | elasticsearch_exporter       |
| Log Monitoring        | Filebeat + Kibana            |

---

## Recommended High Priority Alerts

These are the most important alerts in production:

1. Cluster RED
2. Node Down
3. Heap > 85%
4. Disk > 90%
5. Flood Stage Watermark
6. No Master
7. OOM Error
8. Unassigned Primary Shard
9. Authentication Attack
10. Thread Pool Rejections

---

