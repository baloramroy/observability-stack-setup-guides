# Uptime Kuma – TCP Port Monitor Configuration SOP

## Purpose

This SOP explains how to create a **TCP Port Monitor** in Uptime Kuma using a practical production-style example.

The goal is to help engineers:

* Configure TCP monitoring correctly
* Understand what value should be entered in each field
* Learn the purpose of every configuration item

---

## Example Scenario

Monitor a production appgw service:

| Item        | Value       |
| ----------- | ----------- |
| Server Name | App01       |
| IP Address  | 10.10.20.15 |
| Service     | APPGW       |
| TCP Port    | 3306        |
| Environment | Production  |

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
TCP Port
```

Description

Checks whether the target TCP port is reachable and accepting connections.

---

## 3. Configure Basic Information

### Field: Friendly Name

Example Value

```text
DC3-PROD-APP01_APPGW_10.10.20.15:3306
```

Description

Human-readable monitor name shown in the dashboard.

#

### Field: Hostname

Example Value

```text
10.10.20.15
```


#

### Field: Port

Example Value

```text
3306
```

Description

TCP port number of appgw service, Uptime Kuma will test.

---

## 4. Configure Monitoring Timing

### Field: Heartbeat Interval

Example Value

```text
60
```

>(Unit: seconds)

Description

Defines how often the TCP connectivity check runs.

#

### Field: Retries

Example Value

```text
3
```

Description

Monitor will try 3 times and be marked DOWN only after 3 consecutive failures.

#

### Field: Heartbeat Retry Interval

Example Value

```text
60
```

>(Unit: seconds)

Description

Wait time between retry attempts after a failed check.

---

## 5. Configure Organization Fields

### Field: Monitor Group

Example Value

```text
Production-Application
```

Description

Logical grouping for easier dashboard organization.

#

### Field: Description

Example Value

```text
Primary Production Application Service.
```


---

## 6. Configure Notifications

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

### Description

- Resends alert after every 3 failed checks while monitor remains DOWN.
- But in our setup we are keeping this value 0.

---

## 7. Save Monitor

Click:

```text
Save
```

Uptime Kuma will immediately begin TCP monitoring.

---

## 8. Example Final Configuration

| Field                    | Example Value                         |
| ------------------------ | ------------------------------------  |
| Monitor Type             | TCP Port                              |
| Friendly Name            | DC3-PROD-APP01_APPGW_10.10.20.15:3306 |
| Hostname                 | 10.10.20.15                           |
| Port                     | 3306                                  |
| Heartbeat Interval       | 60 sec                                |
| Retries                  | 3                                     |
| Heartbeat Retry Interval | 60 sec                                |
| Timeout                  | 10 sec                                |
| Monitor Group            | Production-Application                |
| Description              | Primary Production Application Service|
| Notification             | Microsoft-Teams-Infra-Alert           |

---

## 9. Expected Monitoring Behavior

With the above configuration:

```text
Every 60 seconds:
→ Uptime Kuma attempts TCP connection to 10.10.20.15:3306
```

If connection fails:

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

## 10. Recommended Production Values

| Environment          | Interval    | Retries | Retry Interval | Timeout |
| -------------------- | ----------- | ------- | -------------- | ------- |
| Critical Services    | 30 sec      | 3       | 30 sec         | 10 sec  |
| Standard Production  | 60 sec      | 3       | 60 sec         | 10 sec  |
| Low Priority Systems | 120–300 sec | 2       | 60 sec         | 10 sec  |

---


