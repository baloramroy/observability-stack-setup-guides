# Uptime Kuma – Ping Monitor Field Explanation

## Purpose

This document explains every field used while creating a **Ping Monitor** in Uptime Kuma.

The goal is to help engineers understand:

* What each field does
* How monitor logic works
* Recommended production values
* How retry and timeout behavior affects alerting

---

## 1. Monitor Basics

### 1.1 Monitor Type → `Ping`

**Purpose**

Checks whether a target server or device is reachable using ICMP (ping).

**Use Cases**

Use Ping monitoring when you only need to verify:

* Network connectivity
* Server reachability
* Basic infrastructure availability

**Limitations**

Ping monitoring does **not** verify:

* Application health
* Service availability
* Open ports
* Web/API functionality

**For complete monitoring, combine Ping with:**

* HTTP/HTTPS monitors
* TCP port monitors
* Prometheus/Node Exporter
* Application-specific monitoring

#

### 1.2 Friendly Name

**Example:**

```text
DC3-DMZ-APPGW01_10.10.10.11
```

**Purpose**

Human-readable monitor name displayed in the dashboard.

**Recommended Naming Standard**

```text
<Location>-<Zone>-<Hostname>_<IP>
```

**Example:**

```text
DC3-DMZ-APPGW01_10.10.10.11
```

**Benefits**

* Easy identification
* Faster troubleshooting
* Better dashboard organization

#

### 1.3 Hostname

Example:

```text
10.10.10.11
```

**Purpose**

Defines the target host to ping.

**Supported Values**

* IP Address
* Hostname
* FQDN (Fully Qualified Domain Name)

**Best Practice**

| Option     | Advantage                  | Disadvantage   |
| ---------- | -------------------------- | -------------- |
| IP Address | Faster, no DNS dependency  | Less flexible  |
| FQDN       | Easier to manage long-term | Depends on DNS |

---

## 2. Monitoring Interval & Retry Logic

### 2.1 Heartbeat Interval → `60 sec`

**Purpose**

Defines how often Uptime Kuma performs the health check.

**Example**

```text
Every 60 seconds → run ping monitor
```

**Recommended Values**

| Environment             | Recommended Interval |
| ----------------------- | -------------------- |
| Critical Infrastructure | 30 sec               |
| Standard Servers        | 60 sec               |
| Low Priority Systems    | 120–300 sec          |

#

### 2.2 Retries → `3`

**Purpose**

Defines how many consecutive failures are allowed before the monitor is marked DOWN.

**Example Logic**

```text
Check 1 → Failed
Check 2 → Failed
Check 3 → Failed

Result → Monitor marked DOWN
```

**Why It Matters**

Retries help prevent false alerts caused by:

* Temporary packet loss
* Network jitter
* Short-lived latency spikes

**Recommended Values**

| Environment          | Recommended Retries |
| -------------------- | ------------------- |
| Stable LAN/DC        | 2–3                 |
| WAN / Internet / DMZ | 3–5                 |

#

### 2.3 Heartbeat Retry Interval → `60 sec`

**Purpose**

Defines the waiting time between retry attempts after a failed check.

**Example**

```text
1st failure → wait 60 sec
2nd failure → wait 60 sec
3rd failure → mark DOWN
```

**Operational Impact**

| Lower Value      | Higher Value        |
| ---------------- | ------------------- |
| Faster detection | Reduced alert noise |
| More sensitive   | More tolerant       |

#

### 2.4 Global Timeout → `10 sec`

**Purpose**

Maximum time allowed for the entire ping check process.

**Behavior**

If the monitor does not complete within 10 seconds:

```text
Ping process stops
Check marked as failed
```

**Important Clarification**

The monitor itself does not immediately mark the host DOWN after timeout.

**Instead:**

1. Current check fails
2. Retry logic begins (if retries remain)
3. Host is marked DOWN only after retry threshold is exceeded

**Recommended Values**

| Environment        | Recommended Timeout |
| ------------------ | ------------------- |
| LAN / Datacenter   | 5–10 sec            |
| Cross-Region / WAN | 10–20 sec           |

---

## 3. ICMP Packet Configuration

### 3.1 Max Packets → `3`

**Purpose**

Defines how many ICMP packets are sent during a single health check.

**Example**

```text
Send 3 ping packets
Evaluate packet responses
Determine monitor result
```

**Operational Impact**

| Lower Value          | Higher Value                 |
| -------------------- | ---------------------------- |
| Faster checks        | Better accuracy              |
| Less network traffic | Better packet-loss detection |

**Recommended Values**

| Environment             | Recommended Value |
| ----------------------- | ----------------- |
| Standard Monitoring     | 3                 |
| Critical Infrastructure | 5                 |

#

### 3.2 Packet Size → `56 bytes`

**Purpose**

Defines the ICMP payload size.

**Default**

```text
56 bytes
```

(Standard Linux ping payload size)

**Use Cases**

| Size        | Purpose             |
| ----------- | ------------------- |
| 56 bytes    | Normal monitoring   |
| 1000+ bytes | MTU/network testing |

#

### 3.3 Per-Ping Timeout → `2 sec`

**Purpose**

Maximum wait time for each ICMP packet reply.

**Example**

```text
Each ping packet waits up to 2 seconds for reply
```

**Recommended Values**

| Environment      | Recommended Timeout |
| ---------------- | ------------------- |
| LAN / Datacenter | 1–2 sec             |
| WAN / Internet   | 2–5 sec             |

#

### 3.4 Numeric Output

**Purpose**

Displays IP addresses instead of resolved hostnames.

**Example**

| Enabled     | Disabled          |
| ----------- | ----------------- |
| 10.10.10.11 | appgw01.dc3.local |

**Notes**

This setting is mostly cosmetic and does not affect monitoring logic.

---

## 4. Organization & Alerting

### 4.1 Monitor Group

**Purpose**

Used to organize monitors logically.

**Example Groups**

* DC3-DMZ
* Production-Apps
* Database-Servers
* Network-Devices

**Benefits**

* Easier dashboard navigation
* Bulk management
* Better visibility

#

### 4.2 Description

**Purpose**

Stores additional operational information about the monitor.

**Example**

```text
Application Gateway server in DMZ handling external traffic routing.
```

**Best Practice**

Include:

* Server role
* Business purpose
* Environment
* Critical notes

#

### 4.3 Notifications

**Purpose**

Defines where alerts are sent.

**Supported Examples**

* Email
* Telegram
* Slack
* Discord
* Webhook

#

### 4.4 Resend Notification if Down X Times

**Purpose**

Controls repeated alert notifications while the monitor remains DOWN.

**Example**

```text
Resend alert every 3 failed checks while host remains DOWN
```

**Benefits**

* Prevents alert flooding
* Maintains operational visibility during extended outages

---

## 5. Failure Detection Timing for Below Example

### Example Configuration

| Setting            | Value  |
| ------------------ | ------ |
| Heartbeat Interval | 60 sec |
| Retries            | 3      |
| Retry Interval     | 60 sec |


### Failure Timeline

```text
T+0s    → First failure
T+60s   → Second failure
T+120s  → Third failure
Result  → Monitor marked DOWN
```

### Approximate Detection Time

```text
~2–3 minutes
```

---

## 6. Recommended Production Configuration

### Critical Infrastructure Example (DMZ/App Gateway)

| Setting            | Recommended Value |
| ------------------ | ----------------- |
| Heartbeat Interval | 30 sec            |
| Retries            | 3                 |
| Retry Interval     | 30 sec            |
| Max Packets        | 3–5               |
| Per-Ping Timeout   | 2 sec             |
| Global Timeout     | 10 sec            |

### Expected Detection Time

```text
~1–1.5 minutes
```

---

## 7. Key Operational Concepts

| Setting  | Purpose                        |
| -------- | ------------------------------ |
| Interval | How often checks run           |
| Retries  | Failure confirmation threshold |
| Timeout  | Maximum wait duration          |
| Packets  | ICMP accuracy level            |

---

