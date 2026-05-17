## Configure Elasticsearch Secure Keystore (ALL Nodes)

### Why Elasticsearch Keystore Is Required

Elasticsearch uses an internal secure keystore to store sensitive secrets such as:

* TLS keystore passwords
* truststore passwords
* cloud credentials
* repository credentials
* API secrets

Sensitive passwords should NEVER be stored directly inside:

```yaml
elasticsearch.yml
```

Instead, store them securely using:

```bash
elasticsearch-keystore
```

This is the recommended production security practice.

---

## Verify Whether PKCS#12 Files Use Passwords

Check whether your `.p12` certificate files are password protected.

### Verify HTTP Certificate

```bash
openssl pkcs12 -info -in /etc/elasticsearch/certs/http/http.p12
```

If prompted:

```text
Enter Import Password:
```

then the certificate requires a password.

#

### Verify Transport Certificate

```bash
openssl pkcs12 -info -in /etc/elasticsearch/certs/transport/elastic-transport-certificates.p12
```

---

## Add Secure Passwords To Elasticsearch Keystore

Run the following commands on ALL Elasticsearch nodes.

#

### Add HTTP SSL Keystore Password

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
```

You will be prompted to enter the HTTP certificate password.


### Add HTTP SSL Truststore Password

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.truststore.secure_password
```

### Add Transport SSL Keystore Password

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
```


### Add Transport SSL Truststore Password

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

---

## Verify Stored Secure Settings

**List configured secure settings:**

```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore list
```

**Expected output example:**

```text
xpack.security.http.ssl.keystore.secure_password
xpack.security.http.ssl.truststore.secure_password
xpack.security.transport.ssl.keystore.secure_password
xpack.security.transport.ssl.truststore.secure_password
```

> NOTE:
> Only setting names are displayed.
> Secret values are never shown.

---

## Secure Elasticsearch Keystore Permissions

**Verify permissions:**

```bash
ls -l /etc/elasticsearch/elasticsearch.keystore
```

**Recommended:**

```text
-rw------- 1 root elasticsearch
```

**Fix permissions if required:**

```bash
chmod 600 /etc/elasticsearch/elasticsearch.keystore
chown root:elasticsearch /etc/elasticsearch/elasticsearch.keystore
```

>[!NOTE]
Although the file permission is 600, Elasticsearch can still access the keystore through its privileged startup and internal secure settings handling mechanism.


---

## Update Elasticsearch TLS Configuration

Update the HTTP SSL section inside:

```bash
/etc/elasticsearch/elasticsearch.yml
```

Replace:

```yaml
# HTTP SSL
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http/http.p12
```

With:

```yaml
# HTTP SSL
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http/http.p12
xpack.security.http.ssl.truststore.path: certs/http/http.p12
```

---

## Important Notes

* Elasticsearch keystore is local to each node
* Secure settings must be added individually on EVERY node
* Elasticsearch automatically reads secure settings during startup
* Never store passwords directly inside:

  * `elasticsearch.yml`
  * shell scripts
  * automation files
  * Git repositories

---

