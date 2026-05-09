# Logstash

### What is Logstash?
Logstash is a powerful log processing engine that collects logs, processes them,and sends data to Elasticsearch for search and analysis.

---

### Why Logstash is needed

Logs comes form the server is a raw logs. Raw logs are usually:

* Unstructured
* Hard to search
* Different formats (syslog, JSON, app logs, CSV, etc.)

Logstash **converts raw logs into structured, searchable data** before storing them in Elasticsearch.

---

### What Logstash actually does

Logstash works using **pipelines**, and each pipeline has **3 stages**:

**1. Input**

- Where data comes from:
  * Beats (Filebeat, Metricbeat)
  * Syslog
  * Files
  * Kafka
  * HTTP
  * Databases

- Example:
  ```text
  Input: application.log
  ```

**2. Filter**

Where data is processed.

- Common operations:
  * Parse logs
  * Extract fields
  * Rename fields
  * Remove noise
  * Convert data types
  * Add tags
  * Enrich data (geo, hostname, etc.)

- Example:

   ```text
   "10.1.1.5 GET /login 200"
   →
   {
   ip: "10.1.1.5",
   method: "GET",
   url: "/login",
   status: 200
   }
   ```

**3. Output**

Where processed data goes.

- Common outputs:

  * Elasticsearch
  * Files
  * Kafka

- Example:

   ```text
   Output: Elasticsearch index
   ```

---

### **Real-world use cases**

1. System logs

   * `/var/log/messages`
   * `/var/log/secure`
   * SSH login tracking

2. Application logs

   * Java, Spring Boot, PHP, Node.js
   * Exception parsing
   * Response time extraction

---

### **Logstash position in ELK Stack**

```
Log Source
   |
   v
Filebeat / App / Syslog
   |
   v
LOGSTASH  ← (parse, filter, transform)
   |
   v
ELASTICSEARCH
   |
   v
KIBANA
```

---