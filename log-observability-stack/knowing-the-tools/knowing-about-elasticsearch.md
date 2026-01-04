## 2. ELK Stack Components

### Elasticsearch

Elasticsearch is a distributed search and analytics engine that stores data and allows fast searching, filtering, and analysis of logs and other data.

---

### Why Elasticsearch is needed

Logs processed by Logstash are structured, but without a proper engine they are still difficult to analyze at scale. Elasticsearch is needed because it:

* Stores huge amounts of data
* Provides very fast search
* Allows real-time analysis
* Scales across multiple servers

Without Elasticsearch, searching logs across many servers would be slow and complex.

---

### What Elasticsearch actually does

Elasticsearch stores data in a structured way using **indexes**, **documents**, and **fields**.\
It mainly performs **4 core functions**:

1. Store data
2. Index data
3. Search data
4. Analyze data

---

### Key Elasticsearch concepts

**1. Index**

- An index is like a **database** that stores related data.
- Example:
   ```text
   logs-2026.01.03
   ```


**2. Document**

- A document is a **single log entry** stored in Elasticsearch (in JSON format).
   - A single log entry = One line in a log file

- If your application gets 1,000 requests per minute:
   - Each request generates one log entry
   - Elasticsearch would store 1,000 documents per minute

- Example:

   ```json
   {
   "timestamp": "2026-01-03T10:15:30",
   "ip": "10.1.1.5",
   "method": "GET",
   "url": "/login",
   "status": 200
   }
   ```

**3. Field**

- A field is a **key-value pair** inside a document.
- Examples:

   * ip
   * status
   * url
   * timestamp

---

### Why Elasticsearch is Fast

Elasticsearch is fast because it:

* Uses indexing instead of full file search
* Stores data in an optimized format
* Searches across nodes in parallel

---

### Why Elasticsearch is Used in ELK Stack

- Centralized log storage
- Fast search across millions/billions of logs
- Scalable and highly available
- Works seamlessly with Kibana, Logstash, and Beats

---

### Real-World Example (Log Use Case)
- Application generates logs
- Filebeat sends logs to Elasticsearch

- Elasticsearch:
  - Indexes each log entry
  - Stores it as a JSON document

- Kibana queries Elasticsearch to:
  - Search errors
  - Create dashboards
  - Monitor system behavior

---

### Elasticsearch Position in ELK Stack

```
Log Source
   |
   v
Filebeat / App / Syslog
   |
   v
LOGSTASH
   |
   v
ELASTICSEARCH  ← (store, index, search, analyze)
   |
   v
KIBANA
```
---