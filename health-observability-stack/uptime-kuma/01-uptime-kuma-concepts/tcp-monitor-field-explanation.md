# Uptime Kuma – TCP Port Monitor Field Explanation

## Purpose

This document explains every field used while creating a **TCP Port Monitor** in Uptime Kuma.

The goal is to help engineers understand:

* What each field does
* How TCP monitoring works
* Recommended production values
* How retry and timeout behavior affects alerting

---

## 1. Monitor Basics

### 1.1 Monitor Type → `TCP Port`

**Purpose**

Checks whether a TCP service is reachable by attempting to establish a TCP connection to a specific port.


**What TCP Monitoring Verifies**

TCP monitoring confirms:

* Server is reachable
* Target TCP port is open
* Service is accepting TCP connections


**Common Use Cases**

| Service       | Port  |
| ------------- | ----- |
| SSH           | 22    |
| HTTP          | 80    |
| HTTPS         | 443   |
| MySQL         | 3306  |
| PostgreSQL    | 5432  |
| Redis         | 6379  |
| Elasticsearch | 9200  |
| Zabbix Agent  | 10050 |


**Limitations**

TCP monitoring does **not** verify:

* Application response correctness
* Login/authentication success
* API functionality
* Service performance
* Business logic health

Example:

```text
TCP 443 open ≠ Website fully working
```

The port may be open while the application is still broken internally.

---


## 1.2 Friendly Name

**Example**

```text
DC3-PROD-DB01_MYSQL_10.10.20.15:3306
```

**Purpose**

Human-readable monitor name displayed in the dashboard.


**Recommended Naming Standard**

```text
<Location>-<Environment>-<Hostname>_<Service>_<IP>:<Port>
```


**Example**

```text
DC3-PROD-DB01_MYSQL_10.10.20.15:3306
```

---

## 1.3 Hostname

**Example**

```text
10.10.20.15
```


**Purpose**

Defines the target server or device.


**Supported Values**

* IP Address
* Hostname
* FQDN (Fully Qualified Domain Name)


**Best Practice**

| Option     | Advantage                   | Disadvantage   |
| ---------- | --------------------------- | -------------- |
| IP Address | Faster, no DNS dependency   | Less flexible  |
| FQDN       | Easier long-term management | Depends on DNS |

---

## 1.4 Port

**Example**

```text
3306
```


**Purpose**

Defines which TCP port Uptime Kuma will attempt to connect to.


**Examples**

| Service | Port |
| ------- | ---- |
| SSH     | 22   |
| HTTPS   | 443  |
| MySQL   | 3306 |
| Redis   | 6379 |


**Important Concept**

TCP monitoring only checks:

```text
Can a TCP connection be established?
```

It does not validate:

```text
Is the application functioning correctly?
```

---

## 2. Monitoring Interval & Retry Logic

### 2.1 Heartbeat Interval → `60 sec`

**Purpose**

Defines how often Uptime Kuma performs the TCP connectivity check.


**Example**

```text
Every 60 seconds → attempt TCP connection
```


**Recommended Values**

| Environment             | Recommended Interval |
| ----------------------- | -------------------- |
| Critical Infrastructure | 30 sec               |
| Standard Servers        | 60 sec               |
| Low Priority Systems    | 120–300 sec          |

#

## 2.2 Retries → `3`

**Purpose**

Defines how many consecutive failures are allowed before the monitor is marked DOWN.


**Example Logic**

```text
Check 1 → Connection failed
Check 2 → Connection failed
Check 3 → Connection failed

Result → Monitor marked DOWN
```


**Why It Matters**

Retries help prevent false alerts caused by:

* Temporary network interruptions
* Service restarts
* Short-lived latency spikes
* Firewall fluctuations

#

## 2.3 Heartbeat Retry Interval → `60 sec`

**Purpose**

Defines the waiting time between retry attempts after a failed check.


**Example**

```text
1st failure → wait 60 sec
2nd failure → wait 60 sec
3rd failure → mark DOWN
```

---

## 3. TCP Monitoring Logic

### 3.1 What Happens During a TCP Check

Uptime Kuma performs:

```text
1. DNS resolution (if hostname used)
2. TCP connection attempt
3. Wait for TCP handshake
4. Evaluate success/failure
```

#

### 3.2 Successful Check

A monitor is marked UP when:

```text
TCP connection successfully established
```

Example:

```text
10.10.20.15:3306 → reachable
```

#

### 3.3 Failed Check

A monitor is marked failed when:

* Port closed
* Service not listening
* Firewall blocks connection
* Host unreachable
* Connection timeout occurs

---

## 4. Organization & Alerting

### 4.1 Monitor Group

**Purpose**

Used to organize monitors logically.


**Example Groups**

* Database-Servers
* Production-Applications
* DMZ-Services
* Infrastructure-Network


**Benefits**

* Easier dashboard navigation
* Better visibility
* Simplified management

#

## 4.2 Description

**Purpose**

Stores operational information about the monitor.


**Example**

```text
Primary MySQL database service for production billing application.
```

**Best Practice**

Include:

* Service purpose
* Business impact
* Environment
* Critical operational notes

#

## 4.3 Notifications

**Purpose**

Defines where alerts are sent.


**Supported Examples**

* Email
* Telegram
* Slack
* Discord
* Webhook

#

## 4.4 Resend Notification if Down X Times

**Purpose**

Controls repeated alert notifications while the monitor remains DOWN.


**Example**

```text
Resend alert every 3 failed checks while service remains DOWN
```


**Benefits**

* Prevents alert flooding
* Maintains outage visibility

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

### Critical Infrastructure Example (Database / API / Core Services)

| Setting            | Recommended Value |
| ------------------ | ----------------- |
| Heartbeat Interval | 30 sec            |
| Retries            | 3                 |
| Retry Interval     | 30 sec            |
| Timeout            | 10 sec            |

---

### Expected Detection Time

```text
~1–1.5 minutes
```

---

## 7. Key Operational Concepts

| Setting  | Purpose                         |
| -------- | ------------------------------- |
| Interval | How often checks run            |
| Retries  | Failure confirmation threshold  |
| Timeout  | Maximum TCP wait duration       |
| Port     | Service endpoint being verified |

---


