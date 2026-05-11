# Uptime Kuma Integration with Microsoft Teams Using Webhook

## Objective

The objective of this SOP is to define the standardized procedure for **integrating Uptime Kuma with Microsoft Teams** using an **Incoming Webhook**.
This enables automated alert notifications from **Uptime Kuma → Microsoft Teams** for real-time incident awareness.

---

## Prerequisites

* **Uptime Kuma** deployed and running.
* Access to **Microsoft Teams** with permission to add connectors.
* A dedicated **Teams channel** for monitoring alerts.
* **Uptime Kuma** web UI access to update **webhook** configuration.

---

## Architecture Overview

### Alert Notification Flow

```
Uptime Kuma
   |
   | (webhook)
   v
Microsoft Teams Channel
```

---

## Create Microsoft Teams Incoming Webhook

### Step 1: Select Teams Channel

* Open **Microsoft Teams**
* Navigate to the desired **Team** channel
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

## Update **Uptime Kuma** Configuration

* First log in to **uptime kuma**
* From the top right conrner click the **drop down** and select **settings**
* Then from the **settings** menu click on **Notifications**
* Now click on **Set Up Notitfication** to configure the **webhook**.
* Now in this new setup page ->
  * Selecet **Microsft Teams** from **Notification Type** dropdown field.
  * Then provide a **Name** for this alert.
  * After that Provide the **Webhook** url created earlier in the **webhook url field**.
  * Now click on **Test** to see alert are coming or not. If it is successfully added then you will see an alert notification on dedicated teams channel.
  * After successful test, click on **Save** to save this **webhook**.
 
--- 
