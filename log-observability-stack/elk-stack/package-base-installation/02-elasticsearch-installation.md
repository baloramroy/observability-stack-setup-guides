## Install Elasticsearch (All Nodes)

### Import Elastic GPG Key

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

#

### Add Elasticsearch Repository

Create e repo file:
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
```

> Note:

For **Elasticsearch 9.x** installation, change the version name only:

```bash
baseurl=https://artifacts.elastic.co/packages/9.x/yum
```

#

### Install Elasticsearch

To Check all Available Elasticsearch Version:
```bash
dnf list --showduplicates elasticsearch
#
dnf info elasticsearch
```
Now install ES from the repo:
```bash
dnf clean all
dnf makecache
dnf install elasticsearch -y
```

⚠ Do NOT start it yet.

---

## Other Installation Guides:
- Previous: [Elasticsearch Node Preparation Prerequisites](01-elasticsearch-node-preparations.md)
- Next: [Elasticsearch Cluster Configuration & Role Assignment](03-elasticsearch-clustar-configuration-and-role-assignment.md)