# Kibana Authentication Method (Service Account Token or kibana_system)

## Purpose

Defines supported **authentication** methods for **Kibana → Elasticsearch secure communication.**

- This SOP describes how to **securely authenticate** Kibana to Elasticsearch using the built-in `kibana_system` user and **Service Account Tokens**.
- Although **Service Account Tokens** are the **recommended authentication method** for **Elasticsearch 8.x and 9.x**.

---

## Supported Methods

1. **Service Account Token**
     - Modern Elastic method (8.x/9.x)
     - No password management
     - Machine-to-machine authentication
2. **kibana_system** User
     - Legacy but fully supported
     - Username/password based authentication


---

## Prerequisites

Before starting, verify:

* Elasticsearch **cluster** is running
* Elasticsearch **security** is enabled
* Elasticsearch **TLS** is configured
* **Kibana** is **installed**
* **Kibana** can reach **Elasticsearch** over **HTTPS**

---



## ⬛ Method 1: Service Account Token




## Architecture

```text
+-----------+           HTTPS            +----------------+
|  Browser  | <-----------------------> |     Kibana     |
+-----------+                           +----------------+
                                                |
                                                | HTTPS
                                                | Service Account Token
                                                v
                                       +----------------+
                                       | Elasticsearch  |
                                       +----------------+
```

---

## Create a Service Account Token

- Run on any Elasticsearch node:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-service-tokens create elastic/kibana kibana-token
  ```

- Example output:

  ```text
  SERVICE_TOKEN elastic/kibana/kibana-token = AAEAAWVsYXN0aWMva2liYW5hL2tpYmFuYS10b2tlbjo...
  ```

>[!IMPORTANT] 
Copy and securely store the generated token.

---

## Service Account Token Validation

After creating the token, verify that Elasticsearch accepts it.

### Verify the Token Exists

- Run:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-service-tokens list elastic/kibana
  ```

- Example:

  ```text
  elastic/kibana/kibana-token
  ```
  > “This only confirms creation, not validity”

#

### Verify Token Works
  
- Example:
  
  ```bash
  curl -k \
  -H "Authorization: Bearer <SERVICE_TOKEN>" \
  https://es-node-1:9200/_security/_authenticate?pretty
  ```

- Expected:
  ```json
  {
    "username" : "elastic/kibana",
    ...
  }
  ```

- This proves:

  - ✓ Token exists
  - ✓ Token is valid
  - ✓ Elasticsearch accepts it
  - ✓ Authentication succeeds

---

## Store Service Account Token in Kibana Keystore (Recommended)

Instead of storing the **token** directly in `kibana.yml`, store it **securely** in the Kibana **keystore**.

### Create the Keystore

- If the keystore does not already exist:

  ```bash
  /usr/share/kibana/bin/kibana-keystore create
  ```

#

### Add the Service Account Token

- Run:

  ```bash
  /usr/share/kibana/bin/kibana-keystore add elasticsearch.serviceAccountToken
  ```

  > Enter the token when prompted.

- Example:

  ```text
  AAEAAWVsYXN0aWMva2liYW5hL2tpYmFuYS10b2tlbjo...
  ```

#

### Verify Keystore Entry

- Run:

  ```bash
  /usr/share/kibana/bin/kibana-keystore list
  ```

- Expected:

  ```text
  elasticsearch.serviceAccountToken
  ```

#

### Configure Kibana

- Use this when configure kibana in the next SOP:

  ```yaml
  elasticsearch.serviceAccountToken: <token>
  ```
- Or
  
  ```
  Defined only in keystore, not in kibana.yml
  ```
> [!NOTE]
> When the token is stored in the **Kibana keystore**, it is **not exposed** in **plaintext** within `kibana.yml`. Kibana automatically reads it **via key name**.


- **Service account token is tied to:**

  - cluster
  - namespace (elastic/kibana)
  - privileges of that service account

---

## Result

- After completing this SOP:

  ```text
  Browser
      │
      ▼
  Kibana
      │
      │ HTTPS + Service Account Token
      ▼
  Elasticsearch
  ```

>[!NOTE]
Kibana **authenticates** securely to **Elasticsearch** using a **dedicated Service Account Token**, which is the **recommended** approach for **Elasticsearch 9**.x.

---

## ⬛ Method 2: **kibana_system** User and Password


## Architecture

```text
+-----------+           HTTPS            +----------------+
|  Browser  | <-----------------------> |     Kibana     |
+-----------+                           +----------------+
                                                |
                                                | HTTPS
                                                | kibana_system
                                                | Username/Password
                                                v
                                       +----------------+
                                       | Elasticsearch  |
                                       +----------------+
```


---

## Verify kibana_system User Exists

The `kibana_system` user is a built-in Elasticsearch user.

- Verify:

  ```bash
  curl -k -u elastic \
  https://192.168.0.124:9200/_security/user/kibana_system?pretty
  ```

- Expected output:

  ```json
  {
    "kibana_system" : {
      "username" : "kibana_system",
      "enabled" : true
    }
  }
  ```

---

## Set or Reset kibana_system Password

If this is a new deployment or the password is unknown, reset it.

### Option A: Interactive Password Reset

- Run:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
  ```

- Example:

  ```text
  This tool will reset the password of the [kibana_system] user.

  New value:
  XXXXXXXXXXXXXXXX
  ```

Record the generated password securely.

#

### Option B: Specify a Password

- Run:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -i
  ```

  Enter the desired password when prompted.

---

## Test kibana_system Authentication

- Verify the credentials before configuring Kibana.

  ```bash
  curl -k -u kibana_system:<password> https://192.168.0.124:9200
  ```

- Expected output:

  ```json
  {
    "name" : "es-node01",
    "cluster_name" : "production"
  }
  ```

- Successful authentication confirms:
  * User is enabled
  * Password is correct
  * Elasticsearch accepts the credentials
  * HTTPS communication is functioning correctly

- Authentication failures indicate an incorrect password or disabled account.

---

## Store the Password in the Kibana Keystore (Recommended)

Instead of **storing the password directly** in `kibana.yml`, use the **Kibana keystore**.

- Create the keystore if it does not exist:

  ```bash
  /usr/share/kibana/bin/kibana-keystore create
  ```

- Add the password:

  ```bash
  /usr/share/kibana/bin/kibana-keystore add elasticsearch.password
  ```

  >Enter the password when prompted.

### Verify Keystore Entry

- Run:

  ```bash
  /usr/share/kibana/bin/kibana-keystore list
  ```

- Expected:

  ```text
  elasticsearch.password
  ```

  > This approach is more secure than storing the password directly in `kibana.yml`.

### Configure Kibana

- Configure `kibana.yml`:

  ```yaml
  elasticsearch.username: "kibana_system"
  ```

- Remove:

  ```yaml
  elasticsearch.password:
  ```

  > Kibana will retrieve the password from the keystore automatically.

---

## Result

After completing this SOP:

```text
Browser
    │
    ▼
Kibana
    │
    │ HTTPS + kibana_system Credentials
    ▼
Elasticsearch
```
>[!Note]
**Kibana authenticates** securely to **Elasticsearch** using the built-in `kibana_system` account while **maintaining encrypted TLS communication** between all **Elastic Stack** components.

---

## Authentication Method Comparison

| Feature                       | Service Account Token | kibana_system User  |
| ----------------------------- | --------------------- | ------------------- |
| Authentication Type           | Token-based           | Username & Password |
| Password Rotation Required    | No                    | Yes                 |
| Machine-to-Machine Credential | Yes                   | No                  |
| Human Login Capability        | No                    | No                  |
| Secret Exposure Risk          | Lower                 | Higher              |
| Elastic Recommended for 8.x   | Yes                   | Supported           |
| Elastic Recommended for 9.x   | Yes                   | Supported           |
| Suitable for New Deployments  | Yes                   | Yes                 |
| Enterprise Preferred          | Yes                   | Sometimes           |

> [!NOTE]
> For new deployments, Elastic recommends using **Service Account Tokens** whenever organizational standards permit.

---

## Important Authentication Rule

> [!IMPORTANT]
> Use **only one authentication method** for Kibana.

### Choose Either:

- Option 1

  ```yaml
  elasticsearch.serviceAccountToken: "<service-account-token>"
  ```
  > Or Token stored in **Kibana keystore**.

- Option 2

  ```yaml
  elasticsearch.username: "kibana_system"
  elasticsearch.password: "<password>"
  ```

  > or password stored in **Kibana keystore**.

### Do not Configure Both

- Never configure:

  ```yaml
  elasticsearch.serviceAccountToken: "<token>"
  elasticsearch.username: "kibana_system"
  ```

- at the same time.
- Kibana should use a single authentication mechanism.

---