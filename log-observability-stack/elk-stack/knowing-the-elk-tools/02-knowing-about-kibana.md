# Kibana

### What is Kiabana?
Kibana is a data visualization and dashboard tool that is used to search, analyze, and visualize data stored in Elasticsearch.

---

### Why Kibana is needed

Data stored in Elasticsearch is powerful but difficult to understand in raw JSON format. Kibana is needed because it:

* Provides a graphical user interface (GUI)
* Makes data easy to explore
* Turns data into charts and dashboards
* Helps understand logs visually

Without Kibana, users would need to query Elasticsearch manually using APIs.

---

### What Kibana actually does

Kibana connects directly to Elasticsearch and allows users to:

1. Search data
2. Filter logs
3. Visualize data
4. Create dashboards

---

### Key Kibana Features

**1. Discover**

Used to:

* Search logs
* Apply filters
* View log details

Example:

```text
status: 500 AND service: auth
```


**2. Visualizations**

Used to create:

* Bar charts
* Line charts
* Pie charts
* Tables

Examples:

* Error count per service
* Requests per minute
* Response time trends


**3. Dashboards**

Dashboards combine multiple visualizations into a single view.

Examples:

* System health dashboard
* Application monitoring dashboard
* Security monitoring dashboard


**4. Discover Patterns and Trends**

Kibana helps identify:

* Error spikes
* Traffic peaks
* Unusual behavior

---

### Real-world use cases

Log monitoring
* View live logs
* Filter errors quickly

---

### Kibana position in ELK Stack

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
ELASTICSEARCH
   |
   v
KIBANA  ← (search, visualize, dashboard)
```

---