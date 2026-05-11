
```yml
global:
  # After it stops receiving the alert from Prometheus.
  # Alertmanager waits resolve_timeout: 1m before marking an alert as RESOLVED
  # Alert is now officially RESOLVED
  # Then alertmanager check "Am I allowed to send an update now?"
  # This is where group_interval comes in
  # group_interval allows update → RESOLVED message sent
  resolve_timeout: 1m


# The directory from which notification templates are read.
templates:
  - "/etc/alertmanager/template/*.tmpl"

# The root route on which each incoming alert enters.
route:

  # A default receiver which will receive the alert and send as notification.
  # If no custome receiver set
  receiver: "msteams-default"

  # Groups alerts together based on these labels
  # Alerts are grouped only if BOTH of these labels are identical:
  # Alerts with same alertname AND same instance will be grouped into one notification
  # So, if same HighCPU alert fires repeatedly for instance: web-01, they'll be grouped
  # But if same HighCPU alert fires repeatedly for instance: web-01, web-02, web-03 they'll be
  # different alert, not grouped together
  # Or If 3 different alerts fire for thesame server (instance: web-01), they'll be separate alert not grouped together.
  group_by: ["alertname","instance"]

  # Initial waiting period after first alert in a new group
  # Waits 30 seconds to see if more related alerts will arrive
  # Prevents immediate notification for single alert when more might come
  # Useful for batch-related alerts
  group_wait: 30s

  # Time between sending updates for an existing alert group for which the alrt has 
  # already sent.

  # If new alerts arrive or old ones resolve within the same group
  # After the first notification, Alertmanager waits 5 minutes before sending an updated 
  # notification for the same group if new alerts join.
  group_interval: 5m

  # How often to re-send unresolved alerts
  # If an alert is still firing after 60 hours, sends another notification
  # Reminds you that the problem still exists
  # Different from group_interval which is for updates, this is for reminders.
  repeat_interval: 60m



  # All the above attributes are inherited by all child routes and can
  # overwritten on each.

  # The child route trees.
  routes:
    # This routes performs a regular expression match on alert labels to
    # catch alerts that are related to a list of services.
    - receiver: "msteams-default"
      matchers:
        - service=~"foo1|foo2|baz"   

      # The service has a sub-route for critical alerts, any alerts
      # that do not match, i.e. severity != critical, fall-back to the
      # parent node and are sent to "msteams-default"
      routes:
        - receiver: "msteam-crital"
          matchers:
            - severity="critical"

    - receiver: "linux-teams"
      matchers:
        - job="all-linux-server"


# Inhibition rules allow to mute a set of alerts given that another alert is
# firing.
# We use this to mute any warning-level notifications if the same alert is
# already critical.
inhibit_rules:
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    # Apply inhibition if the alertname is the same.
    # CAUTION:
    #   If all label names listed in `equal` are missing
    #   from both the source and target alerts,
    #   the inhibition rule will apply!
    equal: [alertname, cluster, service]


# receivers (plural) is the top-level configuration item that defines a list of receivers
# The template we have discussed is only about microsoft teams template
# Template may differ for different platform configurations

receivers:
  - name: "linux-teams"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/XXXXXXXX/IncomingWebhook/YYYYYYYY"
        send_resolved: true # Notify when alerts are resolved (default)

        # If template use with custome title and text naming convention 
        # Then we definately have to mention the custome title and text convention here
        # It Will not detect the custome template automatically
        # And it will presents the content what's in the custome template only
        title: '{{ template "msteamsv2.custome.title" . }}'
        text: '{{ template "msteamsv2.custome.text" . }}'

  - name: "msteam-crital"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/XXXXXXXX/IncomingWebhook/YYYYYYYY"
        send_resolved: true # Notify when alerts are resolved (default)

        # If custome template use with default title and text naming convention 
        # then we don't have to mention the title and text here.
        # It Will detect the template automatically
        # But it will presents the content what's in the custome template only
        # Not from the alartmanager built in defult template
        title: '{{ template "msteamsv2.default.title" . }}'  # Default title template
        text: '{{ template "msteamsv2.default.text" . }}'  # Default text template


  # if no route matches then alert send from this route
  - name: "msteams-default"
    msteamsv2_configs:
      - webhook_url: "https://outlook.office.com/webhook/XXXXXXXX/IncomingWebhook/YYYYYYYY"
        send_resolved: true # Notify when alerts are resolved (default)
        
        # Default title template
        # title: '{{ template "msteamsv2.default.title" . }}'

        # Default text template  
        # text: '{{ template "msteamsv2.default.text" . }}'
        
        # If we don't use them then,
        # It Will detect the template automatically
        # This will send alerts to Teams using Alertmanager's built-in default templates.
        # Default Title: Shows alert status and name
        # Default Text: Shows all alert labels, annotations, and a summary

tracing:
  endpoint: localhost:4317
  insecure: true
  sampling_fraction: 1.0

```