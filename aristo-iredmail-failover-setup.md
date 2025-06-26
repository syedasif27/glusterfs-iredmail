# iRedMail High Availability Failover Setup

This guide sets up iRedMail with failover support using GlusterFS for shared storage, LDAP Master-Slave replication, and MariaDB Master-Master replication.

## Environment Details

* **Server 1:** `10.10.11.155` - `aristo1.fcoos.in`
* **Server 2:** `10.10.11.156` - `aristo2.fcoos.in`

---

## 1. GlusterFS Setup

### Step 1: Install GlusterFS on both servers

```bash
sudo apt update
sudo apt install -y glusterfs-server
sudo systemctl enable --now glusterd
```

### Step 2: Configure hosts file (both servers)

```bash
echo "10.10.11.155 aristo1.fcoos.in" | sudo tee -a /etc/hosts
echo "10.10.11.156 aristo2.fcoos.in" | sudo tee -a /etc/hosts
```

### Step 3: Create a trusted pool (run on one server)

```bash
gluster peer probe aristo2.fcoos.in
gluster peer status
```

### Step 4: Create and start volume

```bash
mkdir -p /data/gluster-vmail/brick1

# Create volume on any one server:
gluster volume create mailvolume replica 2 \
  aristo1.fcoos.in:/data/gluster-vmail/brick1 \
  aristo2.fcoos.in:/data/gluster-vmail/brick1 \
  force
gluster volume start mailvolume
```

### Step 5: Mount GlusterFS volume

```bash
mkdir -p /data/var/vmail
mount -t glusterfs aristo1.fcoos.in:/mailvolume /data/var/vmail
```

Add to `/etc/fstab`:

```bash
aristo1.fcoos.in:/mailvolume /data/var/vmail glusterfs defaults,_netdev 0 0
```

---

## 2. iRedMail Installation (on both servers) without reboot

### Step 1: Download and run installer

```bash
wget https://github.com/iredmail/iRedMail/archive/refs/tags/1.6.9.tar.gz
tar -xvzf 1.6.9.tar.gz
cd iRedMail-1.6.9/
sudo bash iRedMail.sh
```

* Choose `/data/var/vmail` as the mail storage directory.
* Use OpenLDAP.
* Set domain: `aristo.fcoos.in`.

### Step 2: Post-install sync

* Sync configuration files and SSL certs between both servers.
* Point both installations to the shared GlusterFS storage.

---

## 3. LDAP Master-Slave Replication

### On Master: `aristo2.fcoos.in`

Ensure in `/etc/ldap/slapd.conf`:

```conf
moduleload syncprov.la
overlay syncprov
syncprov-checkpoint 100 10
syncprov-sessionlog 200
```

ACL for replication user:

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: to * by dn="cn=vmail,dc=aristo1,dc=fcoos,dc=in" read by * break
```

### On Slave: `aristo1.fcoos.in`

Add to `/etc/ldap/slapd.conf`:

```conf
syncrepl rid=001
  provider=ldap://10.10.11.156:389
  searchbase="dc=aristo,dc=fcoos,dc=in"
  bindmethod=simple
  binddn="cn=vmail,dc=aristo,dc=fcoos,dc=in"
  credentials="<password>"
  schemachecking=on
  type=refreshAndpersist
  retry="60 +"
  scope=sub
  interval=00:00:01:00
  attrs="*,+"
```

Restart slapd on both:

```bash
systemctl restart slapd
```

---

## 4. MariaDB Master-Master Replication

### Step 1: Edit MariaDB Config

#### Server 1 `/etc/mysql/my.cnf` 10.10.11.155

```ini
[mysqld]
#Master-Master

server-id                   = 1
    log_bin                 = /var/log/mysql/mariadb-bin.log
    log-slave-updates
    log-bin-index           = /var/log/mysql/log-bin.index
    log-error               = /var/log/mysql/error.log
    relay-log               = /var/log/mysql/relay.log
    relay-log-info-file     = /var/log/mysql/relay-log.info
    relay-log-index         = /var/log/mysql/relay-log.index
    auto_increment_increment = 10
    auto_increment_offset   = 1
    binlog_do_db            = amavisd
    binlog_do_db            = iredadmin
    binlog_do_db            = roundcubemail
    binlog_do_db            = sogo
    binlog-ignore-db=test
    binlog-ignore-db=information_schema
    binlog-ignore-db=mysql
    binlog-ignore-db=iredapd
    log-slave-updates
    replicate-ignore-db=test
    replicate-ignore-db=information_schema
    replicate-ignore-db=mysql
    replicate-ignore-db=iredapd

```

#### Server 2 `/etc/mysql/my.cnf`  10.10.11.156

```ini
[mysqld]
#Master-Master

server-id                   = 2
    log_bin                 = /var/log/mysql/mariadb-bin.log
    log-slave-updates
    log-bin-index           = /var/log/mysql/log-bin.index
    log-error               = /var/log/mysql/error.log
    relay-log               = /var/log/mysql/relay.log
    relay-log-info-file     = /var/log/mysql/relay-log.info
    relay-log-index         = /var/log/mysql/relay-log.index
    auto_increment_increment = 10
    auto_increment_offset   = 1
    binlog_do_db            = amavisd
    binlog_do_db            = iredadmin
    binlog_do_db            = roundcubemail
    binlog_do_db            = sogo
    binlog-ignore-db=test
    binlog-ignore-db=information_schema
    binlog-ignore-db=mysql
    binlog-ignore-db=iredapd
    log-slave-updates
    replicate-ignore-db=test
    replicate-ignore-db=information_schema
    replicate-ignore-db=mysql
    replicate-ignore-db=iredapd

```

### Step 2: Create replication user (both servers)

```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'Sys1adm';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

### Step 3: Check master status on both

```sql
SHOW MASTER STATUS;
```

### Step 4: Configure each server as a slave of the other for according to the master's status

On Server aristo1:

```sql
CHANGE MASTER TO
  MASTER_HOST='10.10.11.156',
  MASTER_USER='replicator',
  MASTER_PASSWORD='Sys1adm',
  MASTER_LOG_FILE='mysql-bin.00000X',
  MASTER_LOG_POS=XXXX;
START SLAVE;
```

On Server aristo2:

```sql
CHANGE MASTER TO
  MASTER_HOST='10.10.11.155',
  MASTER_USER='replicator',
  MASTER_PASSWORD='Sys1adm',
  MASTER_LOG_FILE='mysql-bin.00000X',
  MASTER_LOG_POS=XXXX;
START SLAVE;
```

### Step 5: Verify

```sql
SHOW SLAVE STATUS\G
```

---

## Final Notes

* Ensure DNS and mail clients support failover (e.g., MX with equal priority).
* Monitor GlusterFS and replication status.
* Consider HAProxy or Keepalived for automatic failover.
* Regularly test failover scenarios.

---

**Maintained by:** FCOOS Technologies Pvt. Ltd.
