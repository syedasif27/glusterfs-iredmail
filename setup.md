## GlusterFS Setup Documentation for iRedMail (Replica 2)

### Goal

Set up GlusterFS with a **replica 2 volume** to store iRedMail's mail data (`/var/vmail`) across two servers for high availability.

---

## Prerequisites

### Server Setup

| Server   | Hostname  | Gluster Brick Path        | Mail Mount   |
| -------- | --------- | ------------------------- | ------------ |
| Server 1 | `server1` | `/glusterfs/brick1/vmail` | `/var/vmail` |
| Server 2 | `server2` | `/glusterfs/brick1/vmail` | `/var/vmail` |

### Requirements

* Both servers should be on the same subnet (or routable)
* Proper hostname resolution (via `/etc/hosts` or DNS)
* SSH access between servers
* GlusterFS installed

---

## 🔹 Steps on Server 1

### 1. Install GlusterFS

```bash
sudo apt update
sudo apt install -y glusterfs-server
sudo systemctl enable --now glusterd
```

### 2. Configure Hostnames

Edit `/etc/hosts`:

```bash
192.168.1.10 server1
192.168.1.11 server2
```

### 3. Create Brick Directory

```bash
sudo mkdir -p /glusterfs/brick1/vmail
```

> This path should be on a large storage partition, e.g., `/data1`.

### 4. Probe Peer

```bash
sudo gluster peer probe server2
sudo gluster peer status
```

### 5. Create and Start Gluster Volume

```bash
sudo gluster volume create gv0 replica 2 \
  server1:/glusterfs/brick1/vmail \
  server2:/glusterfs/brick1/vmail

sudo gluster volume start gv0
```

### 6. Mount Volume to `/var/vmail`

```bash
sudo mkdir -p /var/vmail
sudo mount -t glusterfs server1:/gv0 /var/vmail
```

Make it persistent:

```
server1:/gv0 /var/vmail glusterfs defaults,_netdev 0 0
```

### 7. Set Permissions

```bash
sudo chown -R vmail:vmail /var/vmail
```

### 8. Install iRedMail

Run the official iRedMail installer:

```bash
chmod +x iRedMail.sh
sudo ./iRedMail.sh
```

Follow the prompts to set up the mail server.

---

## 🔹 Steps on Server 2

### 1. Install GlusterFS

```bash
sudo apt update
sudo apt install -y glusterfs-server
sudo systemctl enable --now glusterd
```

### 2. Configure Hostnames

Edit `/etc/hosts`:

```bash
192.168.1.10 server1
192.168.1.11 server2
```

### 3. Create Brick Directory

```bash
sudo mkdir -p /glusterfs/brick1/vmail
```

> Same path and layout as Server 1

### 4. Mount Volume to `/var/vmail`

```bash
sudo mkdir -p /var/vmail
sudo mount -t glusterfs server1:/gv0 /var/vmail
```

Make it persistent:

```
server1:/gv0 /var/vmail glusterfs defaults,_netdev 0 0
```

### 5. Set Permissions

```bash
sudo chown -R vmail:vmail /var/vmail
```

### 6. Manually Install iRedMail Components

```bash
sudo apt install -y postfix dovecot-core dovecot-imapd dovecot-lmtpd dovecot-mysql \
  amavisd-new clamav clamav-daemon spamassassin policyd-spf \
  iredapd roundcube roundcube-core roundcube-mysql nginx mariadb-client
```

> Replace `nginx` with `apache2` if using Apache.

### 7. Sync Configuration Files from Server 1

Use `rsync` or `scp`:

```bash
rsync -avz /etc/postfix/ user@server2:/etc/postfix/
rsync -avz /etc/dovecot/ user@server2:/etc/dovecot/
rsync -avz /etc/amavis/ user@server2:/etc/amavis/
rsync -avz /etc/clamav/ user@server2:/etc/clamav/
rsync -avz /etc/spamassassin/ user@server2:/etc/spamassassin/
rsync -avz /opt/iredapd/ user@server2:/opt/iredapd/
rsync -avz /etc/iredapd/ user@server2:/etc/iredapd/
rsync -avz /etc/nginx/sites-enabled/ user@server2:/etc/nginx/sites-enabled/
rsync -avz /etc/roundcubemail/ user@server2:/etc/roundcubemail/
rsync -avz /var/www/roundcubemail/ user@server2:/var/www/roundcubemail/
```

### 8. Enable and Start Services

```bash
sudo systemctl enable --now postfix dovecot amavis clamav-daemon spamassassin iredapd nginx
```

### 9. Test Functionality

* Send a test email via Roundcube
* Check `/var/log/mail.log` and related logs

---

## 🔹 Optional: Verify Gluster Replication

```bash
# On server1
echo "testmail" > /var/vmail/test.txt

# On server2
cat /var/vmail/test.txt
# Output should be: testmail
```

---

## Notes

* Do **not** run the iRedMail installer on Server 2
* GlusterFS replicates data in real-time; ensure network stability
* Don't use a Gluster mount as a brick
* Both servers must have enough disk space for full data set
