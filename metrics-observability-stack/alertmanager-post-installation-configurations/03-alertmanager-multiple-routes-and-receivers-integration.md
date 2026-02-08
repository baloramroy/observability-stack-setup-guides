**Multiple child routs and receiver for each of those routes:**

```yml
global:
  resolve_timeout: 1m

route:
  receiver: "default-route"

  group_by:
    - alertname
    - instance

  group_wait: 30s
  group_interval: 5m
  repeat_interval: 60m

  routes:
    # -------------------------------
    # Critical alerts (highest priority)
    # -------------------------------
    - matchers:
        - severity="critical"
      receiver: "critical-alerts"
      group_wait: 10s
      repeat_interval: 30m

    # -------------------------------
    # Warning alerts
    # -------------------------------
    - matchers:
        - severity="warning"
      receiver: "warning-alerts"
      repeat_interval: 2h

    # -------------------------------
    # Windows servers alerts
    # -------------------------------
    - matchers:
        - job="windows-servers"
      receiver: "windows-alerts"

    # -------------------------------
    # Linux servers alerts
    # -------------------------------
    - matchers:
        - job="linux-servers"
      receiver: "linux-alerts"


# =========================================================
# Receivers
# =========================================================

receivers:
  # -------------------------------
  # Default receiver (fallback)
  # -------------------------------
  - name: "default-route"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/DEFAULT/IncomingWebhook/XXXX"
        send_resolved: true

  # -------------------------------
  # Critical alerts receiver
  # -------------------------------
  - name: "critical-alerts"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/CRITICAL/IncomingWebhook/YYYY"
        send_resolved: true

  # -------------------------------
  # Warning alerts receiver
  # -------------------------------
  - name: "warning-alerts"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/WARNING/IncomingWebhook/ZZZZ"
        send_resolved: true

  # -------------------------------
  # Windows servers receiver
  # -------------------------------
  - name: "windows-alerts"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/WINDOWS/IncomingWebhook/AAAA"
        send_resolved: true

  # -------------------------------
  # Linux servers receiver
  # -------------------------------
  - name: "linux-alerts"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/LINUX/IncomingWebhook/BBBB"
        send_resolved: true

```