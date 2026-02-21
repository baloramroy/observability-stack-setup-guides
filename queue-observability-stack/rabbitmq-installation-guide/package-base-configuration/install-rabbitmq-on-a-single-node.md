# Install RabbitMQ on a Single Node using Package Manager

## Check OS Version

```bash
cat /etc/os-release
```

If RHEL 8/9 or compatible → continue.

---

## Install Required Dependencies

- RabbitMQ needs **Erlang**.

**Important:**\
Do NOT install random Erlang from default repo — use RabbitMQ official repo.

---

## Download RPM Package Directly

**Download Erlang first**

```bash
wget --content-disposition "https://packagecloud.io/rabbitmq/erlang/packages/el/8/erlang-24.3.4.11-1.el8.x86_64.rpm/download.rpm?distro_version_id=205"
```

**Then Download RabbitMQ Server**

```bash
wget --content-disposition "https://packagecloud.io/rabbitmq/rabbitmq-server/packages/el/8/rabbitmq-server-3.9.27-1.el8.noarch.rpm/download.rpm?distro_version_id=205"
```

**Then we can install using yum and dnf like this**

```bash
yum install erlang-24.3.4.11-1.el8.x86_64.rpm
yum install rabbitmq-server-3.9.27-1.el8.noarch.rpm
```
**Notes:** 
- This way we can download the exact version
- Then using yum or dnf we will install the package and resolve dependencies by the package manager.
- We will install `erlang` first then `rabbitmq-server`. 

---

## Or We can Add RabbitMQ Official Repository

**We need:**

- Erlang official repo (older version)
- RabbitMQ official repo

#

**Import GPG keys:**

```bash
rpm --import https://packagecloud.io/rabbitmq/erlang/gpgkey
rpm --import https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey
```

#

**Add Erlang repo:**

```bash
tee /etc/yum.repos.d/rabbitmq-erlang.repo <<EOF
[rabbitmq-erlang]
name=RabbitMQ Erlang Repository
baseurl=https://packagecloud.io/rabbitmq/erlang/el/8/$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://packagecloud.io/rabbitmq/erlang/gpgkey
sslverify=1
EOF
```

**If using EL 8 or 9 change:**

```
el/8 → el/9
```

#

**Add RabbitMQ repo:**

```bash
tee /etc/yum.repos.d/rabbitmq.repo <<EOF
[rabbitmq-server]
name=RabbitMQ Server Repository
baseurl=https://packagecloud.io/rabbitmq/rabbitmq-server/el/8/$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey
sslverify=1
EOF
```

>Again change to `el/9` if needed.


---

## Install Exact Erlang Version

**Check available versions:**

```bash
yum --showduplicates list erlang
```

Look for: `erlang-24.3.4.11`

**Then install:**

```bash
yum install erlang-24.3.4.11
```

**Verify Erlang installed or not:**

```bash
erl -version
```

Expected: `Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 13.1.5`


---

## Install Exact RabbitMQ Version

**Check available versions:**

```bash
yum --showduplicates list rabbitmq-server
```
Look for: `rabbitmq-server-3.9.27`

**Then install:**

```bash
yum install rabbitmq-server-3.9.27
```

---

## Enable and Start RabbitMQ

**Enable rabbitmq**
```bash
systemctl enable rabbitmq-server
```

**Start Rabbitmq**

```bash
systemctl start rabbitmq-server
```

**Check status:**

```bash
systemctl status rabbitmq-server
```

---

## Check RabbitMQ and Erlang Version


**Verify Installed RabbitMQ Versions**

```bash
rabbitmqctl version
```

**Check Erlang version used by RabbitMQ:**
```bash
grep -i "erlang" /var/log/rabbitmq/rabbit@*.log
```

**You should see:**

```
RabbitMQ version: 3.9.27
Erlang: 24.3.4.11
```

---

## Lock Version So It Doesn't Upgrade

**To prevent accidental upgrade:**

```bash
dnf install 'dnf-command(versionlock)' -y
dnf versionlock add rabbitmq-server
dnf versionlock add erlang
```

---

## Enable Management Plugin

**Login to your RabbitMQ server and run:**

```bash
rabbitmq-plugins enable rabbitmq_management
```

**After enabling, restart RabbitMQ:**

```bash
systemctl restart rabbitmq-server
```

**Check Plugin status**
```bash
rabbitmq-plugins list | grep management
```
**Output:** You should see something like: `[E*] rabbitmq_management`

---

## Firewall Configurations

### Check if Port 15672 is Listening

**RabbitMQ Web UI runs on:**

- Port: 15672
- Protocol: HTTP

**Check:**

```
ss -tulnp | grep 15672
```

You should see it listening.

#

### Open Firewall

**If you are using firewalld:**

```bash
firewall-cmd --permanent --add-port=15672/tcp 
firewall-cmd --reload
```

---

## Create an Admin User for Access it from another machine

**By default, RabbitMQ creates a user:**

- username: guest
- password: guest

> BUT: The guest user can ONLY login from localhost.

If you are accessing from another **machine**

**create a new **admin** user:**

```bash
rabbitmqctl add_user admin password
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

---

## Now Access web UI

### If accessing locally

From browser 

```
http://localhost:15672
```

Default login:
  - user: guest
  - pass: guest
>⚠️ Guest works only from localhost.

#

### If accessing from another machine:

From browser

```
http://192.168.1.41:15672
```

Login using credential created earlier:
- admin
- password

---

# 10. Important Directories (RPM Installation Layout)

Since you're working on system-level understanding:

- Binary: `/usr/sbin/rabbitmq-server`
- Config: `/etc/rabbitmq/`
- Data: `/var/lib/rabbitmq/`
- Logs: `/var/log/rabbitmq/`
- Systemd unit: `/usr/lib/systemd/system/rabbitmq-server.service`

This confirms it is installed via RPM.

---


