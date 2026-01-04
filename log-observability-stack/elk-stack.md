
# 1. What is ELK Stack

The **ELK Stack** is a popular **log management and analytics platform** used to collect, process, store, search, and visualize logs and events from systems and applications.

ELK stands for:
1. Elasticsearch
2. Logstash
3. Kibana

It is widely used in **DevOps, SRE, and security monitoring**.


---

### 2.4 Beats (Filebeat – Most Common)

* **Lightweight data shippers**
* Installed on **source servers**
* Sends logs and metrics to:

  * **Elasticsearch**, or
  * **Logstash**
* Common Beat types:

  * **Filebeat** – log files
  * **Metricbeat** – system metrics
  * **Auditbeat** – audit data
  * **Heartbeat** – uptime monitoring

---

### 3. Simple ELK Data Flow (High Level)

```
Application / System Logs
        ↓
     Filebeat
        ↓
   Logstash (optional)
        ↓
   Elasticsearch
        ↓
      Kibana
```

---

