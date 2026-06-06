# Appendix A – Generate Kibana HTTPS Certificate Using OpenSSL

## Purpose of This Appendix

- This appendix describes how to generate a **TLS certificate** for Kibana HTTPS using **OpenSSL**.
- The generated certificate will be used to **secure browser communication** with **Kibana** on port `5601`.
- After **completion**, the communication flow will be:

  ```text
  Browser → HTTPS → Kibana (openssl-generated TLS)
  ```

## Scope:

This appendix covers:

* Creating a **dedicated Kibana Certificate Authority (CA)**
* Creating a **Kibana server private key**
* Creating a **Certificate Signing Request (CSR)**
* Configuring **Subject Alternative Names (SAN)**
* **Signing** the certificate
* **Validating** the certificate
* **Deploying** the certificate to **Kibana**

This appendix does NOT cover:

* Kibana installation
* Kibana configuration
* Elasticsearch TLS
* Elasticsearch certificate generation

---

## TLS Architecture

For browser trust, Kibana requires:

```text
kibana-ca
    │
    └── kibana-server.crt
```

Components:

| File              | Purpose                   |
| ----------------- | ------------------------- |
| kibana-ca.key     | CA private key            |
| kibana-ca.crt     | CA public certificate     |
| kibana-server.key | Kibana private key        |
| kibana-server.csr | Certificate request       |
| kibana-server.crt | Kibana server certificate |

---

## Create Working Directory

Perform certificate generation on a secure administration server.

Create working directory:

```bash
mkdir -p /root/kibana-certificates
cd /root/kibana-certificates
```


## Create Kibana Certificate Authority (CA)

### Generate CA Private Key

```bash
openssl genrsa -out kibana-ca.key 4096
```

- Verify:

  ```bash
  ls -l kibana-ca.key
  ```

  Expected:

  ```text
  -rw------- kibana-ca.key
  ```

#

### Generate CA Certificate

```bash
openssl req \
-x509 \
-new \
-nodes \
-key kibana-ca.key \
-sha256 \
-days 825 \
-out kibana-ca.crt
```

Example values:

```text
Country Name: BD
State: Dhaka
Locality: Dhaka
Organization: Linux Onboard
Organizational Unit: IT
Common Name: kibana-ca
```

#

### Verify CA Certificate

```bash
openssl x509 -in kibana-ca.crt -text -noout
```

Verify:

* Issuer
* Subject
* Validity period
* Public key information

Expected:

```text
Subject: CN=kibana-ca
Issuer: CN=kibana-ca
```

> A CA certificate is self-signed.

---

## Create Kibana Certificate

### Create Kibana SAN Configuration

- Modern browsers require Subject Alternative Names (SAN).
- The Common Name (CN) alone is no longer sufficient.

#

Create SAN Configuration File

```bash
vim kibana-san.cnf
```

Example:

```ini
[req]
default_bits = 2048
default_md = sha256
prompt = no
distinguished_name = dn
req_extensions = req_ext

[dn]
C=BD
ST=Dhaka
L=Dhaka
O=Linux Onboard
OU=IT
CN=kibana-node

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = kibana-node
DNS.2 = kibana.local
IP.1 = 192.168.0.123
```

#

### SAN Planning

- Include every hostname users may use, Like:

  | Access Method        | Include SAN |
  | -------------------- | ----------- |
  | kibana-node          | Yes         |
  | kibana.example.local | Yes         |
  | 192.168.0.123        | Yes         |
  | load balancer DNS    | Yes         |

- If a **hostname or IP** is not included in **SAN**, browsers will **report certificate errors**.

#

###  Generate Kibana Private Key

- Generate private key:

  ```bash
  openssl genrsa -out kibana-server.key 2048
  ```

- Verify:

  ```bash
  ls -l kibana-server.key
  ```

- Expected:

  ```text
  -rw------- kibana-server.key
  ```

#

### Create Certificate Signing Request (CSR)

- Generate CSR:

  ```bash
  openssl req \
  -new \
  -key kibana-server.key \
  -out kibana-server.csr \
  -config kibana-san.cnf
  ```

- Verify CSR

  ```bash
  openssl req \
  -in kibana-server.csr \
  -text \
  -noout
  ```

- Verify:

  * Subject
  * Public Keys
  * SAN entries

- Example:

  ```text
  DNS:kibana-node
  DNS:kibana.local
  IP Address:192.168.0.123
  ```

---

### Sign Kibana Certificate

- Sign the CSR using the Kibana CA.

  ```bash
  openssl x509 \
  -req \
  -in kibana-server.csr \
  -CA kibana-ca.crt \
  -CAkey kibana-ca.key \
  -CAcreateserial \
  -out kibana-server.crt \
  -days 3650 \
  -sha256 \
  -extfile kibana-san.cnf \
  -extensions req_ext
  ```

- Generated files:

  ```text
  kibana-server.crt
  kibana-server.key
  ```

#

### Verify Kibana Certificate

**Inspect certificate:**

```bash
openssl x509 \
-in kibana-server.crt \
-text \
-noout
```

**Verify:**

- Subject

  ```text
  CN=kibana-node
  ```

- Issuer

  ```text
  CN=kibana-ca
  ```

- SAN

  ```text
  DNS:kibana-node
  DNS:kibana.example.local
  IP Address:192.168.0.123
  ```

- Validity

  ```text
  Not Before
  Not After
  ```

#

### Verify Certificate Chain

- Verify certificate against CA:

  ```bash
  openssl verify \
  -CAfile kibana-ca.crt \
  kibana-server.crt
  ```

- Expected:

  ```text
  kibana-server.crt: OK
  ```

---

## Deploy Certificates to Kibana Server

- Create certificate directory on **Kibana Node**:

  ```bash
  mkdir -p /etc/kibana/certs/kibana-certs
  ```

- Copy files:

  ```bash
  scp /root/kibana-certificate/kibana-server.crt root@192.168.0.123:/etc/kibana/certs/kibana-certs/

  scp /root/kibana-certificate/kibana-server.key root@192.168.0.123:/etc/kibana/certs/kibana-certs/

  scp /root/kibana-certificate/kibana-ca.crt root@192.168.0.123:/etc/kibana/certs/kibana-certs/
  ```

---

## Configure Ownership & Permissions

Run on `kibana-node`:

- Set Ownership

  ```bash
  chown -R root:kibana /etc/kibana/certs
  ```

- Set Permissions

  ```bash
  find /etc/kibana/certs -type d -exec chmod 750 {} \;
  find /etc/kibana/certs -type f -exec chmod 640 {} \;
  ```

---

## Validate Permissions

- Verify path permissions:

  ```bash
  namei -l /etc/kibana/certs/kibana-certs/kibana-server.key
  ```

- Confirm:

  * root owns files
  * kibana group has read access
  * others have no access


---

## Browser Trust Configuration

- Browsers must trust:

  ```text
  kibana-ca.crt
  ```

  Otherwise users will receive certificate warnings.

#

### Windows

- Import:

  ```text
  kibana-ca.crt
  ```

- Into:

  ```text
  Trusted Root Certification Authorities
  ```

- Process using MMC:

  This method provides a visual interface to manage certificates.

  - Press `Windows + R`, type `mmc`, and press **Enter**.
  - In the console, go to `File > Add/Remove Snap-in`.
  - Select `Certificates` from the list and click `Add >`.
  - Choose **Computer** account, then **Next**. Select **Local Computer**, click **Finish**, and then **OK**.
  - In the left panel, expand `Certificates (Local Computer)` > `Trusted Root Certification Authorities`.
  - Right-click on **Certificates**, go to **All Tasks**, and select **Import....**
  - Click **Next**, browse to your **certificate** file, and follow the **prompts**. Ensure the wizard places it in the `Trusted Root Certification Authorities store`.


#

### RHEL / Rocky / AlmaLinux

- Copy Certificate in this path:

  ```bash
  cp kibana-ca.crt \
  /etc/pki/ca-trust/source/anchors/
  ```

- Then Run:

  ```bash
  update-ca-trust
  ```

#

### Ubuntu

- Copy Certificate in this path:

  ```bash
  cp kibana-ca.crt \
  /usr/local/share/ca-certificates/
  ```

- Then Run:

  ```bash
  update-ca-certificates
  ```

---

## Security Recommendations

**Protect the following files:**

```text
kibana-ca.key
kibana-server.key
```

**Never:**

* Email private keys
* Store private keys in Git repositories
* Share private keys between environments

**Recommended:**

* Store CA private keys offline
* Maintain certificate inventory
* Document expiration dates
* Rotate certificates before expiry

---

## Generated Files Summary

| File              | Description               |
| ----------------- | ------------------------- |
| kibana-ca.key     | CA private key            |
| kibana-ca.crt     | CA public certificate     |
| kibana-server.key | Kibana private key        |
| kibana-server.csr | Certificate request       |
| kibana-server.crt | Kibana server certificate |
| kibana-ca.srl     | CA serial number file     |

---

## Appendix Completion

- This appendix successfully generated:

  * Kibana Certificate Authority
  * Kibana server certificate
  * Kibana private key
  * SAN-enabled certificate
  * Browser-trusted certificate chain

- The generated files can now be used in:

  ```yaml
  server.ssl.enabled: true
  server.ssl.certificate: /etc/kibana/certs/kibana-certs/kibana-server.crt
  server.ssl.key: /etc/kibana/certs/kibana-certs/kibana-server.key
  ```

- within the main **Kibana Secure Configuration & Elasticsearch Integration SOP**.
