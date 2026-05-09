
After the cluster is up for the first time, reset the elastic superuser password to ensure security.

## Get Elastic Password

**Elasticsearch 8.x or later version auto-generates password.**

Retrieve it:

```bash
grep "generated password" /var/log/elasticsearch/elasticsearch.log
```

OR reset:

```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

>You will be prompted for custome password

**Verify login**

```bash
curl -u elastic https://192.168.10.11:9200
```
>You will get a valid json response

---

## Verify Cluster Staus

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

For 3-node cluster: `Aim for GREEN`

#

### Check ES Nodes Membership

From any node:

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

## Remove Initial Bootstrap Setting

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

---

## Restart One-by-One

Restart safely:

Node1:

```bash
systemctl restart elasticsearch
```

>Wait healthy.

Then:

* Node2
* Node3

---

## Final Verification

Check Cluster Health:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cluster/health?pretty
```

Expected:

```json
"status" : "green"
```

Check node membership:

```bash
curl -k -u elastic https://192.168.10.11:9200/_cat/nodes?v
```
---
