# Install Elasticsearch using Package Manager

## Supported OS

This guide is intended for:
- RHEL 8/9
- Rocky Linux 8/9
- AlmaLinux 8/9
- CentOS Stream 8/9

---

## Installation Options

You have **two options** for installing the **Elasticsearch RPM package**:
1. **[From the RPM Repository](#1-install-from-the-rpm-repository)**
2. **[Manual Installation](#2-download-and-install-elasticsearch-rpm-manually)**

#

## 1. Install from the RPM repository

### Import Elastic GPG Key

Elasticsearch signs all of their packages with the Elasticsearch signing key with fingerprint:

```bash
Fingerprint:
4609 5ACC 8548 582C 1A26 99A9 D27D 666C D88E 42B4
```

Download and install the public signing key:

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

#

### Define a Repository for Elasticsearch

Create a repo file:
```bash
vi /etc/yum.repos.d/elasticsearch.repo
```

Add these lines:

```ini
[elasticsearch]
name=Elasticsearch repository
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
type=rpm-md
```

Varify Repository:
```bash
dnf repolist | grep elasticsearch
```

**Note:**

For **Elasticsearch 9.x** installation, change the version name only:

```bash
baseurl=https://artifacts.elastic.co/packages/9.x/yum
```


#

### Install Elasticsearch from the repository you defined earlier

To Check all Available Elasticsearch Version:

```bash
dnf list --showduplicates elasticsearch
#
dnf info elasticsearch
```

Now install Elasticsearch from the repository:

```bash
dnf clean all
dnf makecache --refresh
dnf install elasticsearch-8.17.3 -y
```

⚠ Do NOT start it yet.

---


## 2. Download and Install Elasticsearch RPM Manually

Manual installation is useful when:

- **Internet access** is restricted
- Installing in **offline** environments
- Installing a **specific Elasticsearch** version
- Maintaining **controlled package deployment**

#

### Download Elasticsearch RPM Package

Download a specific Elasticsearch RPM package version:

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm
```

Example:

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.17.3-x86_64.rpm
```

#

### Download the SHA512 Checksum File

Download the checksum file used for package integrity verification:

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm.sha512
```

Example:

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.17.3-x86_64.rpm.sha512
```

#

### Verify Package Integrity

**Verify** that the **RPM** package was **downloaded correctly** and was not **modified** or corrupted.

Run:

```bash
sha512sum -c elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm.sha512
```

Example:

```bash
sha512sum -c elasticsearch-8.17.3-x86_64.rpm.sha512
```

Expected output:

```bash
elasticsearch-8.17.3-x86_64.rpm: OK
```

If verification fails:

- Re-download the package
- Check internet/proxy issues
- Verify file corruption

#

### Install Elasticsearch RPM Package

Install the RPM package:

```bash
rpm -ivh elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm
```

Example:

```bash
rpm -ivh elasticsearch-8.17.3-x86_64.rpm
```

OR using DNF (recommended because it resolves dependencies automatically):

```bash
dnf install elasticsearch-8.17.3-x86_64.rpm
```

#

### Download and Install the RPM in One Step:

Replace <SPECIFIC.VERSION.NUMBER> with the **Elasticsearch version number** you want. For example, you can replace <SPECIFIC.VERSION.NUMBER> with 9.0.0.

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm.sha512
sha512sum -c elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm.sha512
sudo rpm --install elasticsearch-<SPECIFIC.VERSION.NUMBER>-x86_64.rpm
```

---

## Verify Elasticsearch Package Installation

- Check whether Elasticsearch was installed successfully:

  ```bash
  rpm -qa | grep elasticsearch
  ```

  OR

  ```bash
  dnf list installed | grep elasticsearch
  ```

- Expected output example:

  ```bash
  elasticsearch.x86_64    8.17.3-1
  ```

---

## Verify Installed Elasticsearch Sevice

- Service Verification
  ```bash
  systemctl status elasticsearch
  ```

---

## Verify Installed Elasticsearch Version

- Check the installed Elasticsearch version:

  ```bash
  /usr/share/elasticsearch/bin/elasticsearch --version
  ```

- Example output:

  ```bash
  Version: 8.17.3
  ```

---

## Important Notes After Installation

After installation:

**i. Elasticsearch service files are created**

- Default configuration directory becomes:

  ```bash
  /etc/elasticsearch/
  ```

- Default data directory:

  ```bash
  /var/lib/elasticsearch/
  ```

- Default log directory:

  ```bash
  /var/log/elasticsearch/
  ```

- Systemd service file:

  ```bash
  /usr/lib/systemd/system/elasticsearch.service
  ```

**ii. Elasticsearch user and group are automatically created**

- Vary User and Group

  ```bash
  id elasticsearch
  ```

- Expected:
  - elasticsearch user
  - elasticsearch group

---

## Do NOT Start Elasticsearch Yet

⚠ Do NOT start Elasticsearch immediately after installation.

First complete:

- System configuration
- Kernel parameter tuning
- JVM configuration
- Cluster configuration
- Security configuration
- Discovery settings
- Memory settings

After all configurations are completed, then start Elasticsearch.

---

## Upgrade Warning

Very important for future readers.

```
WARNING:
Do not upgrade Elasticsearch nodes randomly in a production cluster.
```

Always follow:
- compatibility matrix
- rolling upgrade procedures
- snapshot backup practices

---

## Other Installation Guides:
- Previous: [Elasticsearch Node Preparation](01-system-preparation.md)
- Next: [Elasticsearch TLS Certificate Generation](03-elasticsearch-tls-certificate-generation.md)