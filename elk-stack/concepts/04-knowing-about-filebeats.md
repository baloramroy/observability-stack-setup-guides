# Filebeat

### What is Filebeat?
Filebeat is a lightweight agent that collects logs from servers and forwards them to Logstash or Elasticsearch.

**Common Beat types:**

  * **Filebeat** – log files
  * **Metricbeat** – system metrics
  * **Auditbeat** – audit data
  * **Heartbeat** – uptime monitoring

---

### **Why Filebeat is needed**

Servers generate logs continuously, and sending these logs directly to Elasticsearch is not always efficient. Filebeat is needed because it:

* Is lightweight and fast
* Uses very low system resources
* Reliably ships logs
* Handles log rotation safely

Without Filebeat, applications would need to send logs themselves.

---


### **What Filebeat actually does**

Filebeat performs **4 main tasks**:

1. Reads log files
2. Tracks file changes
3. Ships new log entries
4. Handles failures and restarts

---

### **Key Filebeat components**

**1. Input**

Defines where logs come from.

Examples:

* `/var/log/messages`
* `/var/log/secure`
* Application log files


**2. Harvester**

A harvester reads a single log file line by line and sends events to Filebeat.


**3. Registry**

Filebeat keeps track of:

* Which files were read
* How much data was already sent

This prevents duplicate logs after restart.


**4. Output**

Defines where logs are sent.

Common outputs:
* Logstash
* Elasticsearch

---


### **Real-world use cases**

1. System logs

   * `/var/log/messages`
   * `/var/log/secure`

2. Application logs

   * Web server access logs
   * Application error logs

3. Container logs

   * Docker logs
   * Kubernetes logs

4. Centralized logging

   * Collect logs from many servers
   * Send to a central ELK stack

---

### **Filebeat position in ELK Stack**

```
Log Source
   |
   v
FILEBEAT  ← (collect, ship)
   |
   v
LOGSTASH / ELASTICSEARCH
   |
   v
KIBANA
```

---