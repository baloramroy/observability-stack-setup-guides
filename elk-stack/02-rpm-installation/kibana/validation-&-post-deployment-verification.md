# Kibana Validation & Post-Deployment Verification

## Purpose of This SOP

This SOP validates that Kibana has been successfully integrated with the secured Elasticsearch cluster and is operating correctly.

This document covers:

* Kibana service validation
* HTTPS validation
* Certificate validation
* Authentication validation
* Browser access validation
* Kibana health validation
* Kibana ↔ Elasticsearch communication validation
* Common troubleshooting procedures

This SOP assumes:

* Kibana installation is complete
* Kibana secure integration is complete
* Kibana service is configured
* Elasticsearch cluster is operational
* TLS certificates are deployed
* Kibana service has been started

---

## Validation Objectives

Successful completion of this SOP confirms:

* Kibana service is running
* Kibana is listening on HTTPS
* Kibana certificate is valid
* Browser access is working
* Kibana APIs are operational
* Elasticsearch communication is functioning correctly

---

## Environment Information

| Hostname    | IP Address    | Role          |
| ----------- | ------------- | ------------- |
| es-node-1   | 192.168.0.124 | Elasticsearch |
| es-node-2   | 192.168.0.125 | Elasticsearch |
| es-node-3   | 192.168.0.126 | Elasticsearch |
| kibana-node | 192.168.0.123 | Kibana        |

---

## Verify Kibana Service Status

- Run on the Kibana server.

  ```bash
  systemctl status kibana
  ```

- Expected:

  ```text
  active (running)
  ```

- Verify:

  * Service is running
  * No startup failures
  * No certificate-related errors

---

## Verify Kibana Service Enablement

- Verify Kibana starts automatically after reboot.

  ```bash
  systemctl is-enabled kibana
  ```

- Expected:

  ```text
  enabled
  ```

---

## Verify Kibana Listening Port

- Verify Kibana is listening on HTTPS port 5601.

  ```bash
  ss -tlnp | grep 5601
  ```

- Expected:

  ```text
  LISTEN 0 511 0.0.0.0:5601
  ```

  or

  ```text
  LISTEN 0 511 [::]:5601
  ```


---

## Verify Kibana HTTPS Certificate

- Run:

  ```bash
  openssl x509 \
  -in /etc/kibana/certs/kibana-certs/kibana-server.crt \
  -text -noout
  ```

- Verify:

  * Subject
  * Issuer
  * SAN entries
  * Validity period
  * Key usage
  * Extended key usage

---

## Verify Certificate Expiration

- Run:

  ```bash
  openssl x509 \
  -enddate \
  -noout \
  -in /etc/kibana/certs/kibana-certs/kibana-server.crt
  ```

- Example:

  ```text
  notAfter=Dec 31 23:59:59 2028 GMT
  ```

- Verify:

  * Certificate has not expired
  * Remaining validity period is acceptable

---

## Verify Certificate and Private Key Match

- Certificate:

  ```bash
  openssl x509 \
  -noout \
  -modulus \
  -in /etc/kibana/certs/kibana-certs/kibana-server.crt | openssl md5
  ```

- Private Key:

  ```bash
  openssl rsa \
  -noout \
  -modulus \
  -in /etc/kibana/certs/kibana-certs/kibana-server.key | openssl md5
  ```

- Expected:

  ```text
  MD5(stdin)= xxxxxxxxxxxx
  MD5(stdin)= xxxxxxxxxxxx
  ```

  Both hashes must match.

---

## Verify Kibana Health API

- Run:

  ```bash
  curl -k https://kibana-node:5601/api/status
  ```

- Expected response:

  ```json
  {
    "overall": {
      "level": "available"
    }
  }
  ```

- Verify:

  * Status API responds
  * Overall status is available

---

## Verify Kibana Server Response

- Run:

  ```bash
  curl -Ik https://kibana-node:5601
  ```

- Expected:

  ```text
  HTTP/1.1 302 Found
  ```

  or

  ```text
  HTTP/1.1 200 OK
  ```

  This confirms Kibana web service is responding.

---

## Review Kibana Startup Logs

View recent logs:

```bash
journalctl -u kibana -n 100
```

Successful startup should contain entries similar to:

```text
Kibana is now available
```

Verify logs contain:

* No TLS errors
* No authentication failures
* No certificate validation failures
* No plugin startup failures

---

## Monitor Kibana Logs in Real Time

Run:

```bash
journalctl -u kibana -f
```

Observe for:

* TLS errors
* Authentication failures
* Elasticsearch connection failures
* Plugin errors

---

## Browser Access Validation

Open:

```text
https://kibana-node:5601
```

Verify:

* Login page loads
* HTTPS lock icon is displayed
* No browser certificate warnings
* Page loads successfully


---

## Verify Kibana ↔ Elasticsearch Integration

After login verify access to:

* Stack Management
* Dev Tools
* Index Management
* Discover
* Dashboards
* Monitoring (if enabled)

Successful access confirms Kibana can communicate with Elasticsearch.

---

## Dev Tools Validation

- Navigate:

  ```text
  Dev Tools
  ```

- Run:

  ```json
  GET /
  ```

- Expected:

  ```json
  {
    "name": "es-node-1"
  }
  ```

  This confirms Elasticsearch API communication.

---

## Cluster Health Validation

- Run:

  ```json
  GET _cluster/health
  ```

- Expected:

  ```json
  {
    "status": "green"
  }
  ```

- Verify:

  * Cluster status is green
  * Cluster information is returned successfully

---

## Index Visibility Validation

- Run:

  ```json
  GET _cat/indices?v
  ```

- Verify:

  * Indices are listed
  * No authorization errors occur

---

## Verify Kibana Saved Objects Service

- Navigate:

  ```text
  Stack Management → Saved Objects
  ```

- Verify:

  * Saved Objects page loads
  * No encryption key warnings appear

---

## Verify Encryption Key Configuration

Review Kibana logs.

- Verify absence of messages similar to:

  ```text
  Generating a random key
  ```

  or

  ```text
  Encryption key is not set
  ```

- Successful validation confirms:

  * Encryption keys are configured correctly
  * Keys persist across restarts

---

## Common Troubleshooting

| Problem                            | Possible Cause                     |
| ---------------------------------- | ---------------------------------- |
| Kibana service fails to start      | Configuration error                |
| Kibana service fails to start      | Incorrect certificate permissions  |
| TLS handshake failure              | Wrong CA certificate               |
| Hostname verification failure      | SAN mismatch                       |
| Unable to connect to Elasticsearch | DNS issue                          |
| Unable to connect to Elasticsearch | Firewall issue                     |
| Authentication failed              | Incorrect kibana_system credential |
| Browser certificate warning        | Browser does not trust CA          |
| Login page unavailable             | Kibana service not running         |
| Saved Objects warnings             | Missing encryption keys            |

---

## Troubleshooting Log Locations

- Kibana Logs

  ```text
  /var/log/kibana/
  ```

  or

  ```bash
  journalctl -u kibana
  ```

- Elasticsearch Logs

  ```text
  /var/log/elasticsearch/
  ```

---

## Operational Readiness Checklist

Verify all items below:

| Validation Item                  | Status |
| -------------------------------- | ------ |
| Kibana service running           | □      |
| Service enabled                  | □      |
| Port 5601 listening              | □      |
| DNS resolution working           | □      |
| Elasticsearch HTTPS reachable    | □      |
| TLS handshake successful         | □      |
| Kibana certificate valid         | □      |
| Browser access working           | □      |
| Login successful                 | □      |
| Dev Tools functional             | □      |
| Cluster health green             | □      |
| Saved Objects accessible         | □      |
| No TLS errors in logs            | □      |
| No authentication errors in logs | □      |

---

## SOP Completion Summary

This SOP validated:

* Kibana service operation
* HTTPS functionality
* Certificate validity
* TLS trust configuration
* DNS functionality
* Authentication functionality
* Kibana API availability
* Browser accessibility
* Elasticsearch integration
* Cluster communication
* Operational readiness

>Successful completion confirms Kibana is securely integrated with Elasticsearch and ready for production use.

---
