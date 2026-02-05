
# Alertmanager Integration with Microsoft Teams Using Webhook


## Objective

The objective of this SOP is to define the standardized procedure for **integrating Alertmanager with Microsoft Teams** using an **Incoming Webhook**.
This enables automated alert notifications from **Prometheus → Alertmanager → Microsoft Teams** for real-time incident awareness.

---

## Prerequisites

* **Alertmanager** deployed and running.
* **Prometheus** integrated with Alertmanager.
* Access to **Microsoft Teams** with permission to add connectors.
* A dedicated **Teams channel** for monitoring alerts.
* **Shell access** (sudo/root) to update Alertmanager configuration.
* Basic understanding of Alertmanager routing and receivers.

---

## Architecture Overview

### Alert Notification Flow

```
Prometheus
   |
   | (fires alerts)
   v
Alertmanager
   |
   | (webhook)
   v
Microsoft Teams Channel
```

---

## Create Microsoft Teams Incoming Webhook

### Step 1: Select Teams Channel

* Open **Microsoft Teams**
* Navigate to the desired **Team**
* Select the **Channel** where alerts should be delivered

#

### Step 2: Add Incoming Webhook Connector

1. From the left upper corner click **⋯ (More options)** next to the channel
2. Select **Manage Channel**
3. Then from the upper top section select **Setting -> Connectors -> Edit**.
4. Search for **Webhook** then click on **Configure**
5. Provide a **Name** (example: `Alert-Observer`)
6. (Optional) Upload a **custom icon**
7. Click **Create**

#

### Step 3: Copy Webhook URL

* Copy the generated **Webhook URL**
* Store it securely (this is a sensitive endpoint)

**Example:**

```
https://outlook.office.com/webhook/XXXXXXXX/IncomingWebhook/YYYYYYYY
```

---

## Update Alertmanager Configuration

### Locate Alertmanager Configuration File

Example path:

```
/home/observer/container/alertmanager/alertmanager_config/alertmanager.yml
```


### Add Microsoft Teams Receiver

Update `alertmanager.yml` as follows:

```yaml
global:
  resolve_timeout: 1m

route:
  receiver: "default-route"
  group_by: ["alertname", "instance"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 60m

receivers:
  - name: "default-route"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/XXXXXXXX/IncomingWebhook/YYYYYYYY"
        send_resolved: true
```

**Important Notes:**
* Replace the **webhook URL** with your actual Teams webhook
* `send_resolved: true` ensures resolve notifications are sent
* `title` and `text` is not necessary for default template.


---


## Reload or Restart Alertmanager

- Restart Alertmanager Container

  ```bash
  docker compose restart alertmanager
  ```

- Verify Container Health

  ```bash
  docker inspect -f '{{.State.Health.Status}}' alertmanager
  ```

---

## Validate Microsoft Teams Integration

### Step 1: Trigger a Test Alert

Add a temporary test rule in **prometheus/prom_config/alert_rules/test_rules.yml** file:

```yaml
groups:
- name: test-alerts
  rules:
  - alert: PrometheusTestAlert
    expr: vector(1)
    for: 10s
    labels:
      severity: critical
    annotations:
      summary: "🔥 Prometheus test alert"
      description: "This is a forced test alert to verify Alertmanager integration."
```

Reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

#

### Step 2: Verify Teams Channel

**Expected Result:**

* First alert appears in **prometheus** then **alertmanager** and then
* Alert appears in **Microsoft Teams channel**
* Alert **title**, **summary**, and **description** visible
* Alert resolves when rule is removed

---

## Troubleshooting

### No Alerts in Teams

* Verify webhook URL is correct
* Check Alertmanager logs:

  ```bash
  docker logs alertmanager
  ```
* Confirm Alertmanager can reach the internet (proxy settings if required)

#

### Alerts Visible in Alertmanager but Not in Teams

* Confirm `route.receiver` matches receiver name
* Verify alert labels match routing rules
* Ensure webhook connector is not deleted in Teams

#

### Duplicate or Noisy Alerts

* Adjust:

  * `group_by`
  * `group_interval`
  * `repeat_interval`
* Use severity-based routing

---

## Notes

* Microsoft Teams webhooks **do not support rich formatting** natively.
* For advanced formatting, use:

  * Alertmanager **custom templates**
  * Teams **Workflow / Power Automate**
* Always protect webhook URLs (treat them like secrets).
* Test alerts after every configuration change.

---

