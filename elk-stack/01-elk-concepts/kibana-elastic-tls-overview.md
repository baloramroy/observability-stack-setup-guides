# Why Kibana used CA trust & Why not its Own Certificate

Right now your confusion is mainly about these two things:

1. Why Kibana only needs to “trust” Elasticsearch CA
2. Why Kibana itself does not always need its own certificate

These are actually two different TLS directions.

---

## First Understand the Communication Direction

When Kibana connects to Elasticsearch:

```text
Kibana  --->  Elasticsearch
(client)      (server)
```

In this connection:

* Elasticsearch acts as TLS SERVER
* Kibana acts as TLS CLIENT

This is exactly like:

```text
Browser ---> HTTPS Website
```

Example:

```text
Chrome ---> https://google.com
```

Here:

* Google server presents certificate
* Browser verifies trust
* Browser itself usually does NOT present a certificate

Same thing happens with Kibana.

---

## Why Elasticsearch Needs Certificate

Because Elasticsearch is the HTTPS server.

When Kibana connects:

```text
https://es-node-1:9200
```

Elasticsearch sends:

* its certificate
* certificate chain
* CA identity

Kibana then verifies:

1. Is this certificate signed by trusted CA?
2. Is hostname valid?
3. Is certificate expired?
4. Is certificate integrity valid?

If valid:

```text
TLS connection established
```

---

## Why Kibana Only Needs CA Trust

- Because Kibana only needs to VERIFY Elasticsearch identity.
- It does NOT need to prove its own identity unless **Mutual TLS (mTLS)** is configured.

  ```text
  Mutual TLS (mTLS)
  ```

  is configured.
- Or Kibana needs to proves its identity to client 
---

## Normal HTTPS (One-Way TLS)

- This is what your SOP currently uses.

  Flow:

  ```text
  Kibana ---> Elasticsearch
  ```

- Elasticsearch sends certificate.
- Kibana verifies certificate.

DONE.

- Only server identity is verified.
- Exactly like browser visiting HTTPS website.

---

## Visual Concept

```text
              [ Elasticsearch ]
                      |
            presents certificate
                      |
                      V
                      |
                  [ Kibana ]
            verifies CA signature
```

Kibana only needs:

```yaml
elasticsearch.ssl.certificateAuthorities:
```

OR

```yaml
elasticsearch.ssl.truststore.path:
```

- because Kibana only needs **TRUST.**

---

## Then Why Is Copying `.p12` Bad in Production?

Because your `.p12` file contains MORE than trust.

Your file:

```text
elastic-stack-ca.p12
```

usually contains:

* CA certificate
* CA PRIVATE KEY

The private key is the dangerous part.

---

## Why CA Private Key Is Extremely Sensitive

Because whoever owns CA private key can generate trusted certificates.

- Meaning attacker could create:

  ```text
  fake-es-node-1 certificate
  fake-kibana certificate
  fake-logstash certificate
  ```

- And all systems would trust them.
- This completely destroys PKI trust.

---

## Production Best Practice

Production systems **NEVER distribute CA private** key.

- Instead distribute ONLY:

  ```text
  CA public certificate
  ```

- Usually:

  ```text
  ca.crt
  ```

  or

  ```text
  http_ca.crt
  ```

---

## Correct Production Architecture

### CA System

- Keep ONLY on secure CA host:

  ```text
  ca.key
  ca.p12
  ```

  - NEVER distribute.

#

### Elasticsearch Nodes

- Receive only:

  * node certificate
  * node private key
  * CA public certificate

- Example:

  ```text
  es-node-1.crt
  es-node-1.key
  ca.crt
  ```

#

### Kibana

- Needs ONLY:

  ```text
  ca.crt
  ```

  - because Kibana only verifies Elasticsearch.

---

## Production Example

- Instead of this:

  ```yaml
  elasticsearch.ssl.truststore.path:
  /etc/kibana/certs/elastic-stack-ca.p12
  ```

- Production usually uses:

  ```yaml
  elasticsearch.ssl.certificateAuthorities:
    - /etc/kibana/certs/http_ca.crt
  ```

- This file contains ONLY:

  ```text
  -----BEGIN CERTIFICATE-----
  ```

  - No private key.
  - Safe to distribute.

---

## Better Production Flow

### On Elasticsearch CA Host

Export ONLY CA certificate:

```bash
openssl pkcs12 -in elastic-stack-ca.p12 -clcerts -nokeys -out http_ca.crt
```

OR if using Elastic-generated HTTP CA:

```bash
cp http_ca.crt /tmp/
```


### Copy ONLY Public CA Cert

```bash
scp http_ca.crt root@kibana-node:/etc/kibana/certs/
```


### Kibana Configuration

```yaml
elasticsearch.ssl.certificateAuthorities:
  - /etc/kibana/certs/http_ca.crt
```

This is production-grade.

---

# Then When Does Kibana Need Its Own Certificate?

Kibana needs its own certificate in TWO cases.


## CASE 1 — Browser → Kibana HTTPS

If users access Kibana securely:

```text
Browser ---> HTTPS ---> Kibana
```

Then Kibana becomes TLS SERVER.

So Kibana must present its own certificate.

Example:

```yaml
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key
```

This secures:

```text
Browser <-> Kibana
```

---

## CASE 2 — Mutual TLS (mTLS)

If Elasticsearch requires client certificate authentication.

Then BOTH sides verify each other.

Flow:

```text
Kibana ---> Elasticsearch
```

Elasticsearch asks:

```text
"Show me your certificate"
```

Then Kibana needs:

```yaml
elasticsearch.ssl.certificate:
elasticsearch.ssl.key:
```

This is mTLS.

Not always required.

Usually advanced/high-security deployments.

---

## Your Current SOP Architecture

Right now your SOP has:

```text
Browser ----HTTP----> Kibana ----HTTPS----> Elasticsearch
```

Only second connection encrypted.

---

## Production Recommended Architecture

Production usually uses:

```text
Browser ----HTTPS----> Kibana ----HTTPS----> Elasticsearch
```

Meaning:

1. Kibana has server certificate
2. Elasticsearch has server certificate
3. Kibana trusts Elasticsearch CA
4. Browser trusts Kibana CA

---

## Extremely Important TLS Mental Model

Always ask:

```text
Who is acting as SERVER?
Who is acting as CLIENT?
```

SERVER:

* presents certificate

CLIENT:

* verifies trust

---

## In Your Environment

### Kibana → Elasticsearch

```text
Elasticsearch = SERVER
Kibana = CLIENT
```

Therefore:

* Elasticsearch needs cert
* Kibana needs CA trust

---

### Browser → Kibana

If HTTPS enabled:

```text
Kibana = SERVER
Browser = CLIENT
```

Therefore:

* Kibana needs cert
* Browser needs CA trust

---

## Final Production Recommendation For Your SOP

Instead of:

```yaml
elasticsearch.ssl.truststore.path:
```

Use:

```yaml
elasticsearch.ssl.certificateAuthorities:
  - /etc/kibana/certs/http_ca.crt
```

And distribute ONLY:

```text
http_ca.crt
```

NOT:

```text
elastic-stack-ca.p12
```

to Kibana.

That is the real production-grade approach.
