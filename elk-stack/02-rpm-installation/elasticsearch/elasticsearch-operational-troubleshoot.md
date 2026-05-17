# Elasticsearch Operational Troubleshooting SOP

## 1. Cluster Health Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_cluster/health?pretty
```

Investigate:

* RED cluster
* YELLOW cluster
* delayed allocation

---

## 2. Node Connectivity Issues

Check:

```bash
curl -k -u elastic https://IP:9200/_cat/nodes?v
```

Investigate:

* missing nodes
* disconnected nodes
* split brain symptoms

---

## 3. Master Election Problems

Check:

```bash
curl -k -u elastic https://IP:9200/_cat/master?v
```

Investigate:

* no elected master
* master flapping
* unstable cluster state

---

## 4. Unassigned Shards Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_cat/shards?v
```

AND

```bash
curl -k -u elastic \
https://IP:9200/_cluster/allocation/explain?pretty
```

Investigate:

* disk watermark
* allocation filtering
* replica failure
* node loss

---

## 5. JVM / Heap Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_nodes/stats/jvm?pretty
```

Investigate:

* high heap usage
* frequent GC
* OOM errors

---

## 6. Disk Usage Troubleshooting

Check:

```bash
df -h
```

AND

```bash
curl -k -u elastic https://IP:9200/_cat/allocation?v
```

Investigate:

* flood stage watermark
* shard allocation blocked
* disk imbalance

---

## 7. TLS / Certificate Troubleshooting

Check:

```bash
openssl s_client -connect IP:9300
```

AND

```bash
curl -vk https://IP:9200
```

Investigate:

* expired certificates
* trust issues
* hostname mismatch

---

## 8. Authentication / Security Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_security/_authenticate?pretty
```

Investigate:

* authentication failures
* disabled security
* permission issues

---

## 9. Elasticsearch Log Analysis

Check:

```bash
journalctl -u elasticsearch -xe
```

Investigate:

* bootstrap checks
* transport errors
* discovery failures
* shard corruption
* GC overhead

---

## 10. Performance Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_cat/thread_pool?v
```

AND

```bash
curl -k -u elastic https://IP:9200/_nodes/hot_threads?pretty
```

Investigate:

* blocked thread pools
* slow indexing
* slow searches
* CPU spikes

---

## 11. Snapshot Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_snapshot?pretty
```

Investigate:

* repository inaccessible
* snapshot failure
* restore failure

---

## 12. Cluster Recovery Troubleshooting

Check:

```bash
curl -k -u elastic https://IP:9200/_cat/recovery?v
```

Investigate:

* stuck recovery
* slow recovery
* replica sync issues

---
