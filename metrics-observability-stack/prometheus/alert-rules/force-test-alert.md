## Force a Test Alert
>[!NOTE]
To temporaily check **alert flow** is working or not, we can create this **dummy** alert rule and trigger **forcefully**.

### Dummy Alert:
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