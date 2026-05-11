# Three Node RabbitMQ Cluster Configuration

## RabbitMQ Cluster Requires:

1. Same **Erlang** version on all nodes
2. Same **RabbitMQ** version on all nodes
3. Same **Erlang cookie** on all nodes
4. Proper **hostname** resolution between nodes
5. Ports open:
   * 5672 → AMQP
   * 15672 → Management UI
   * 25672 → Cluster communication
   * 4369 → epmd

---

## Architecture Understanding

Cluster Nodes:

   * 10.x.x.41
   * 10.x.x.42
   * 10.x.x.43

Cluster Type: 3-node cluster\
Recommended queue type: **Quorum Queues (Production recommended)**

---

## Pre Checks - Do This On all 3 Nodes

### Set Hostname

On 41:

```bash
hostnamectl set-hostname rabbit41
```

On 42:

```bash
hostnamectl set-hostname rabbit42
```

On 43:

```bash
hostnamectl set-hostname rabbit43
```

#

### Configure /etc/hosts on ALL nodes

Add This Entry:

```
10.x.x.41 rabbit41
10.x.x.42 rabbit42
10.x.x.43 rabbit43
```

Test from `rabbit41`:

```
ping rabbit42
ping rabbit43
```

**Note:** Must resolve.

---

## Download RPM Package Directly

Download Erlang first

```bash
wget --content-disposition "https://packagecloud.io/rabbitmq/erlang/packages/el/8/erlang-24.3.4.11-1.el8.x86_64.rpm/download.rpm?distro_version_id=205"
```

Then Download RabbitMQ Server

```bash
wget --content-disposition "https://packagecloud.io/rabbitmq/rabbitmq-server/packages/el/8/rabbitmq-server-3.9.27-1.el8.noarch.rpm/download.rpm?distro_version_id=205"
```

Install Using YUM and DNF on All Nodes

```bash
yum install erlang-24.3.4.11-1.el8.x86_64.rpm
yum install rabbitmq-server-3.9.27-1.el8.noarch.rpm
```

**Notes:** 

- This way we can download the **exact** version
- Then using **yum** or **dnf** we will install the package and **resolve dependencies** by the package manager.
- We will install `erlang` first then `rabbitmq-server`.
- If we want **installation** using **add repository** step then we can follow **install-rabbitmq-on-a-single-node.md** tutorial from this folder.

---

## Enable and Start RabbitMQ on All Nodes

Enable rabbitmq
```bash
systemctl enable rabbitmq-server
```

Start Rabbitmq

```bash
systemctl start rabbitmq-server
```

Check status:

```bash
systemctl status rabbitmq-server
```

---

## Check RabbitMQ and Erlang Version


Verify Installed RabbitMQ Versions

```bash
rabbitmqctl version
```

Check Erlang version used by RabbitMQ:
```bash
grep -i "erlang" /var/log/rabbitmq/rabbit@*.log
```

You should see:

```
RabbitMQ version: 3.9.27
Erlang: 24.3.4.11
```

---


## Configure Erlang Cookie

RabbitMQ nodes authenticate using a secret cookie.

### Find cookie:
```bash
cat /var/lib/rabbitmq/.erlang.cookie
```

#

### If Cookie Exist in This Folder

- Pick node1 (41) as master.
- master node cookie - FUAODUOVEFGBLPYBOCPC
- Copy cookie from 41 → 42 and 43

On 42 & 43 Server:

```bash
systemctl stop rabbitmq-server
echo "FUAODUOVEFGBLPYBOCPC" > /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```

**Note:** Copy cookie manually and paste same value.\
⚠ Permissions MUST be 400.

Verify Permission:

```bash
ls -l /var/lib/rabbitmq/.erlang.cookie
```

#

### If not Exists, Create Manually.

Create **Same** cookie on all 3 servers:

```bash
echo "MY_SECRET_COOKIE_123" > /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
```

⚠ Permissions MUST be 400.

Verify Permission:

```bash
ls -l /var/lib/rabbitmq/.erlang.cookie
```

---

## Open Required Firewall Ports

On ALL nodes:

```bash
firewall-cmd --permanent --add-port=5672/tcp
firewall-cmd --permanent --add-port=15672/tcp
firewall-cmd --permanent --add-port=25672/tcp
firewall-cmd --permanent --add-port=4369/tcp
firewall-cmd --reload
```

Ports meaning:

- 5672 → AMQP
- 15672 → Management UI
- 25672 → Cluster communication
- 4369 → epmd

Verify:

```bash
firewall-cmd --list-ports
```

---

## Join Node to Cluster

### On Node `rabbit42`:

Stop node (if running):

```bash
rabbitmqctl stop_app
```

Reset Rabbitmq-server:

```bash
rabbitmqctl reset
```

Join cluster to master node:

```bash
rabbitmqctl join_cluster rabbit@rabbit41
```

Start app:

```bash
rabbitmqctl start_app
```

#

### On Node `rabbit43`:

Run Following Command:

```bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbit41
rabbitmqctl start_app
```

---


## Verify Cluster

On any node:

```bash
rabbitmqctl cluster_status
```

You should see:

```
rabbit@rabbit41
rabbit@rabbit42
rabbit@rabbit43
```

---


## Enable Management Plugin

On rabbit41:

```bash
rabbitmq-plugins enable rabbitmq_management
systemctl restart rabbitmq-server
rabbitmq-plugins list | grep management
```

**Output:** You should see something like: `[E*] rabbitmq_management`

Access from Browser:

```
http://rabbit41:15672
Or
http://10.x.x.41:15672
```

Default:

* user: guest
* pass: guest

⚠ guest works only locally. 

Create Admin user to Access it from other machine:

```bash
rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

Delete guest (Optional):

```bash
rabbitmqctl delete_user guest
```

---


## Production HA Policy

Use quorum queues (recommended modern approach):

```
rabbitmqctl set_policy quorum-queues "^" \
'{"queue-type":"quorum"}' \
--apply-to queues
```

This ensures queues are replicated across nodes.

---

## Final Checklist

* Same Erlang version
* Same RabbitMQ version
* Same cookie
* Hostname resolution works
* Firewall open
* Service enabled
* Quorum policy applied
* guest user removed