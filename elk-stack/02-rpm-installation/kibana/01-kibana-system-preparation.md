# Kibana Installation Prerequisites & System Configurations

## Kibana System Requirements Clarifications

### Kibana Does Not Require Linux kernel tuning

Unlike **Elasticsearch**, Kibana does NOT require:

  * vm.max_map_count
  * bootstrap checks
  * memory locking (mlockall)
  * mmap tuning
  * JVM tuning (heap sizing, GC tuning)

### Why?

* Kibana is a **Node.js-based application**
* It does **NOT store or search data locally**
* It only **communicates with Elasticsearch** via **HTTP/HTTPS** API

So all Elasticsearch-level kernel tuning is NOT applicable here.

---

## Optional System Tuning (Recommended for Production)

Kibana is lightweight, but for **stability** in production environments, you may apply: `File Descriptor Limit`

>[!Note]
Temporary ulimit changes are not recommended for system services.

### Permanent File Descriptor Setting for Kibana

Since Kibana runs as a **systemd service**, you must set limits via **systemd override**, not shell.


- Create systemd override directory

  ```bash
  mkdir -p /etc/systemd/system/kibana.service.d
  ```

- Create override config

  ```bash
  vi /etc/systemd/system/kibana.service.d/override.conf
  ```

  Add:

  ```ini
  [Service]
  LimitNOFILE=65535
  ```

- Reload systemd daemon & Restart Kibana

  ```bash
  systemctl daemon-reload
  systemctl restart kibana
  ```

- Verify setting

  ```bash
  systemctl show kibana | grep LimitNOFILE
  ```

  Expected:

  ```text
  LimitNOFILE=65535
  ```



>[!Note]
However, RPM/systemd installation usually handles this **automatically**. So in most cases, **manual tuning is NOT required**.

---

## Java Requirement Clarification

Kibana does not depend on **Java runtime**.

### Explanation:

  * Kibana runs on **Node.js runtime (bundled with installation)**
  * No **external Java** installation is required
  * No **JVM configuration** is needed

### Conclusion:

  * You do NOT need to install Java for Kibana at all.

---

## Version Compatibility (CRITICAL RULE)

Kibana and Elasticsearch MUST **match in version**.

### Rule:

```text
Elasticsearch 9.x  →  Kibana 9.x
Elasticsearch 8.x  →  Kibana 8.x
```
* Minor version mismatch can also cause API incompatibility.

### Why this is important:

* API compatibility depends on exact version alignment
* Security features (TLS, tokens, APIs) are version-specific
* Mismatch will cause Kibana startup failure or connection errors

---

## Verify the Installed Elastic Versions (Prerequisite Check)

Before installation, confirm the installed elasticsearch versions:

```bash
rpm -qa | grep elasticsearch
```

OR:

```bash
dnf list installed | grep elasticsearch
```

Output:

```bash
elasticsearch-9.4.1-1.x86_64
```

This ensures:

* Correct major version of elasticsearch
* Compatibility with kibana

---