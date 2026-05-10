# Elasticsearch Cluster Roles Explanation

Elasticsearch is a **distributed system**, and roles define *who does what* inside the cluster.

#

## 1. Master Node (Cluster Brain)

### Responsibilities

- Maintains **cluster state** (metadata)
- Decides **which node holds which shard**
- Handles:
  * Node join/leave
  * Index creation/deletion
  * Shard allocation & rebalancing

### What it DOES NOT do

- No heavy **search queries**
- No **indexing** workload
- No **data storage**

### Important Design Rules

- Always use **dedicated master nodes in production**
- Minimum: **3 master nodes (quorum)**
- Keep them:
  * Low CPU usage
  * Stable (no overload)

### Why?

- If master is unstable → **cluster becomes unstable**

---

## 2. Data Node (Worker)

### Responsibilities

- Stores **actual data (indices & shards)**
- Executes:
  * Search queries
  * Aggregations
  * Indexing operations

### Types (Advanced Concept)

-  **Hot nodes** → frequently written data
-  **Warm nodes** → less frequently accessed
-  **Cold nodes** → archival data

### Important Design Rules

- High RAM
- Fast disk (SSD recommended)
  * Scaling data nodes = **scaling performance**

### Key Insight

- Data nodes are the **core workload carriers** of Elasticsearch.

---

## 3. Ingest Node (Preprocessing Engine)

### Responsibilities

- Processes data **before indexing** using pipelines

### Example Tasks

- Parse logs (e.g., Apache/Nginx logs)
- Convert fields
- Add metadata (timestamps, geo info)
- Drop unwanted fields

### Example Pipeline Flow

```
Filebeat → Ingest Node → Data Node
```

### When to Use

- Heavy log transformation
- Replacing Logstash (light use cases)
- You do NOT need **Logstash** if your **log processing is simple** and can be handled by **Ingest pipelines.**
- But logstash is still important for **advanced, scalable, enterprise pipelines.**

### Important Design Rules

- Can be combined with data nodes (small clusters)
- Use **dedicated ingest nodes** if:
    * High ingestion rate
    * Complex pipelines

---

## 4. Coordinating Node (Smart Router)

### Reality Check

**Every node is a coordinating node by default**

### Responsibilities

- Accepts client requests
- Sends request to relevant data nodes
- Collects & merges results
- Returns final response


### Important Design Rules

In large clusters:

- Use **dedicated coordinating nodes**
  * Helps reduce load on data nodes

---

## Role Interaction

### Indexing Flow

```
Client → Coordinating Node → Ingest Node → Data Node
```

### Search Flow

```
Client → Coordinating Node → Data Nodes → Coordinating Node → Client
```

---

## Architecture Insight

### Small Cluster (like 3-node setup)

You can combine roles:

| Node   | Roles                  |
| ------ | ---------------------- |
| Node 1 | master                 |
| Node 2 | master + data + ingest |
| Node 3 | master + data + ingest |

>This is not **balanced and practical**

### For "Balanced and Practical" Setup Cluster Like This

| Node   | Roles                  |
| ------ |------------------------|
| Node 1 | master + data + ingest |
| Node 2 | master + data + ingest |
| Node 3 | master + data + ingest |

>All three nodes share the load equally. **This** is balanced and practical for a small cluster.

---

## Large Production Cluster

Separate roles:

| Role         | Dedicated Nodes |
| ------------ | --------------- |
| Master       | 3 nodes         |
| Data         | Many nodes      |
| Ingest       | Optional        |
| Coordinating | Optional        |

---

## Common Mistakes

- Putting **data + master on same overloaded node**
- Having only **1 master node** → split brain risk
- Ignoring ingest load → indexing bottleneck
- No coordinating nodes in large clusters

---

## Simple Memory Trick

Think like a company:

- **Master Node** → Manager
- **Data Node** → Workers
- **Ingest Node** → Data processor
- **Coordinating Node** → Receptionist

---
