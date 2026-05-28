## 4. Kibana Configuration (MOST IMPORTANT PART)

---

### 4.1 Main Configuration File

Edit:

```bash
vi /etc/kibana/kibana.yml
```

---

### 4.2 Basic Kibana Configuration

Add / modify the following:

```yaml
server.port: 5601
server.host: "0.0.0.0"

server.name: "kibana-node"

elasticsearch.hosts:
  - "http://es-node-1:9200"
  - "http://es-node-2:9200"
  - "http://es-node-3:9200"

kibana.index: ".kibana"
```

Explanation:
• `0.0.0.0` → allow browser access
• Multiple ES hosts → high availability
• Kibana auto-switches if one ES node fails

---

## 5. Firewall Configuration (Kibana Node)

Open Kibana port:

```bash
firewall-cmd --permanent --add-port=5601/tcp
firewall-cmd --reload
```

---

## 6. Start Kibana

---

### 6.1 Enable & Start Service

```bash
systemctl daemon-reexec
systemctl enable kibana
systemctl start kibana
```

---

### 6.2 Check Status

```bash
systemctl status kibana
```

Expected:

```text
Active: active (running)
```

If it fails:

```bash
journalctl -u kibana -f
```

---

## 7. Access Kibana UI

From your **browser**:

```
http://192.168.10.20:5601
```

You should see:
• Kibana loading screen
• Welcome page

If page doesn’t load:
• Check firewall
• Check Kibana logs
• Check ES cluster health

---

## 8. Verify Kibana ↔ Elasticsearch Connection

In Kibana UI:
• Go to **Stack Management**
• Elasticsearch should show **Connected**

Or via CLI on Kibana node:

```bash
curl http://es-node-1:9200
```

---

## 9. Common Problems & Fixes

### Kibana not starting

• ES cluster down
• Wrong ES hostnames
• Firewall blocked
• Memory too low

---

### Browser shows “Kibana server is not ready yet”

• ES not reachable
• ES still starting
• Check:

```bash
journalctl -u kibana -f
```

---

## 10. What You Have Achieved

Now you have:
• 3-node Elasticsearch cluster
• Dedicated Kibana node
• HA connection to ES
• Production-style separation

This is **exactly how real ELK environments are built**.

---