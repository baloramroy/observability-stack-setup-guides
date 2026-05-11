# Uptime Kuma – Ping Monitor Configuration SOP

## Purpose

This SOP explains how to create a **Ping Monitor** in Uptime Kuma using a practical production-style example.

The goal is to help engineers:

* Configure Ping monitoring correctly
* Understand what value should be entered in each field
* Learn the purpose of every configuration item

---

## Example Scenario

Monitor a production application gateway server:

| Item        | Value       |
| ----------- | ----------- |
| Server Name | APPGW01     |
| IP Address  | 10.10.10.11 |
| Environment | Production  |
| Zone        | DMZ         |

---

## 1. Open Uptime Kuma

Login to Uptime Kuma dashboard.

Click:

```text
Add New Monitor
```

---

## 2. Select Monitor Type

### Field: Monitor Type

Value to Select

```text
Ping
```

Description

Checks whether the target server or device is reachable using ICMP ping.

---

## 3. Configure Basic Information

### Field: Friendly Name

Example Value

```text
DC3-DMZ-APP01_10.10.10.11
```

Description

Human-readable monitor name shown in the dashboard.

#

### Field: Hostname

Example Value

```text
10.10.10.11
```

Description

Target server IP address or DNS hostname to ping.

---

## 4. Configure Monitoring Timing

### Field: Heartbeat Interval

Example Value

```text
60
```

>(Unit: seconds)

Description

Defines how often the ping check runs.

#

### Field: Retries

Example Value

```text
3
```

Description

Monitor will be marked DOWN only after 3 consecutive failed checks.

#

### Field: Heartbeat Retry Interval

### Example Value

```text
60
```

>(Unit: seconds)

Description

Wait time between retry attempts after a failed check.

#

### Field: Global Timeout

Example Value

```text
10
```

>(Unit: seconds)

Description

Maximum time allowed for the **entire ping proces**s before marking the check as failed.

---

## 5. Configure ICMP Packet Settings

### Field: Max Packets

Example Value

```text
3
```

Description

Number of ICMP packets sent during each health check.

#

### Field: Packet Size

Example Value

```text
56
```

>(Unit: bytes)

Description

Defines the ICMP payload size used during ping checks.

#

### Field: Per-Ping Timeout

Example Value

```text
2
```

>(Unit: seconds)

Description

Maximum wait time for each individual ping packet reply.

#

### Field: Numeric Output

Example Value

```text
Enabled
```

Description

Displays **IP address** instead of **resolved hostname** in monitor output.

---

# 6. Configure Organization Fields

### Field: Monitor Group

Example Value

```text
Production-DMZ
```

Description

Logical grouping for easier dashboard organization.

#

### Field: Description

Example Value

```text
Production application gateway server handling external traffic routing.
```

---

## 7. Configure Notifications

### Field: Set Up Notification

Example Value

```text
Microsoft-Teams-Infra-Alert
```

Description

Defines where alert notifications will be sent.

N.B: [Notification and Webhook Setup Guide](microsoft-teams-webhook-integration-in-uptime-kuma.md)

#

### Field: Resend Notification if Down X Times

Example Value

```text
3
```

Description

- Resends alert after every 3 failed checks while the monitor remains DOWN.
- But in our setup we are keeping this value 0.

---

## 8. Save Monitor

Click:

```text
Save
```

Uptime Kuma will immediately begin Ping monitoring.

---

## 9. Example Final Configuration

| Field                    | Example Value                         |
| ------------------------ | ------------------------------------- |
| Monitor Type             | Ping                                  |
| Friendly Name            | DC3-DMZ-APP01_10.10.10.11             |
| Hostname                 | 10.10.10.11                           |
| Heartbeat Interval       | 60 sec                                |
| Retries                  | 3                                     |
| Heartbeat Retry Interval | 60 sec                                |
| Global Timeout           | 10 sec                                |
| Max Packets              | 3                                     |
| Packet Size              | 56 bytes                              |
| Per-Ping Timeout         | 2 sec                                 |
| Numeric Output           | Enabled                               |
| Monitor Group            | Production-DMZ                        |
| Description              | Production application gateway server |
| Notification             | Microsoft-Teams-Infra-Alert           |

---

## 10. Expected Monitoring Behavior

With the above configuration:

```text
Every 60 seconds:
→ Uptime Kuma sends ICMP ping requests to 10.10.10.11
```

If ping fails:

```text
1st failure → retry
2nd failure → retry
3rd failure → monitor marked DOWN
```

Approximate outage detection time:

```text
~2–3 minutes
```

---

## 11. Recommended Production Values

| Environment             | Interval    | Retries | Retry Interval | Global Timeout |
| ----------------------- | ----------- | ------- | -------------- | -------------- |
| Critical Infrastructure | 30 sec      | 3       | 30 sec         | 10 sec         |
| Standard Production     | 60 sec      | 3       | 60 sec         | 10 sec         |
| Low Priority Systems    | 120–300 sec | 2       | 60 sec         | 10 sec         |

---

## 12. Operational Best Practice

Use Ping monitoring for:

* Basic server reachability
* Network connectivity verification
* Infrastructure availability monitoring
* Initial outage detection

Do NOT rely only on Ping monitoring for service health.

Always combine with:

* TCP Port monitoring
* HTTP/HTTPS monitoring
* Prometheus metrics
* Log monitoring
* Application health checks
