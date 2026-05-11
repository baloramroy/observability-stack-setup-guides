# Custome Template integration for Microsoft Teams v2 

## How MS Teams v2 Works in Alertmanager

With `msteamsv2_configs`, you can customize:

- `title`
- `text`
- `summary`

These fields support Go templates.

So we will:

1. Create **template** file
2. **Load** it in `alertmanager.yml`
3. Reference it inside `msteamsv2_configs` webhook

---

## Create Template Directory and Files

First Create a directory for Storing all the templates
```bash
mkdir -p /home/observer/container/alertmanager/alertmanager_config/templates
```

Create Production-Ready MS Teams v2 Template file:

```bash
vim /home/observer/container/alertmanager/alertmanager_config/templates/msteams.tmpl
```


Paste this inside `msteams.tmpl`:

```yaml
{{ define "teams.title" }}
{{ if eq .Status "firing" }}
[FIRING] {{ .CommonLabels.alertname }}
{{ else }}
[RESOLVED] {{ .CommonLabels.alertname }}
{{ end }}
{{ end }}


{{ define "teams.message" }}

**Information**

- Severity: {{ .CommonLabels.severity }}

{{ range .Alerts }}
----------------------------------------------------
- Job: {{ .Labels.job }}
- Instance: {{ .Labels.instance }}
- Summary: {{ .Annotations.summary }}
- Description: {{ .Annotations.description }}

**Time**
- StartsAt: {{ .StartsAt }}
{{ if eq $.Status "resolved" }}
- EndsAt: {{ .EndsAt }}
{{ end }}

{{ end }}
{{ end }}
```

This template:

- Handles **firing** and **resolved** separately
- Works with **grouped** alerts


---

## How It Looks in Teams

When firing:

![Alert Firing Template](/metrics-observability-stack/images/alertmanager/template-images/template1-firing.png)

When resolved:

![Alert Firing Template](/metrics-observability-stack/images/alertmanager/template-images/template1-resolve.png)

---

## Modify alertmanager.yml

Open alertmanager configuration file:

```bash
vim /home/observer/container/alertmanager/alertmanager_config/alertmanager.yml
```

Add template path at top level:

```yaml
templates:
  - "/etc/alertmanager/templates/*.tmpl"
```
>Note: This is refer to container inside directory


Then configure receiver:

```yaml
receivers:
  - name: "default-route"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/XXXXXXXX"
        send_resolved: true
        title: '{{ template "teams.title" . }}'
        text: '{{ template "teams.message" . }}'
```

Important:
- `'{{ template "teams.title" . }}'`
- `.` passes full alert context to template.

---

## Restart or Reload Alertmanager

If systemd:

```bash
systemctl restart alertmanager
```

Or reload:

```bash
docker restart alertmanager
```

---

## Validate Before Restart

Always check config:

```bash
amtool check-config /etc/alertmanager/alertmanager.yml
```

If template syntax is wrong → Alertmanager will fail.

---

