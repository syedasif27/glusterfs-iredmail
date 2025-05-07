# glusterfs-iredmail

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

## Step-by-Step Instructions

### 1. Install GlusterFS on Both Servers

```bash
sudo apt update
sudo apt install -y glusterfs-server
```

Start and enable the Gluster service:

```bash
sudo systemctl enable --now glusterd
```

---

### 2. Configure Hostnames

Edit `/etc/hosts` on both servers:

```bash
# Example (adjust IPs accordingly)
192.168.1.10 server1
192.168.1.11 server2
```

---

### 3. Create Brick Directories

```bash
# On server1
sudo mkdir -p /data/brick1/vmail

# On server2
sudo mkdir -p /data1/brick1/vmail
```

> These must be empty and ideally on dedicated partitions (e.g., 1.5 TB disks).

---

### 4. Probe Peers (Run on server1)

```bash
sudo gluster peer probe server2
```

Check peer status:

```bash
sudo gluster peer status
```

---

### 5. Create Gluster Volume (Run on server1)

```bash
sudo gluster volume create gv0 replica 2 \
  server1:/data/brick1/vmail \
  server2:/data1/brick1/vmail
```

(Optional) Add `transport tcp` explicitly:

```bash
sudo gluster volume create gv0 replica 2 transport tcp \
  server1:/data/brick1/vmail \
  server2:/data1/brick1/vmail
```

---

### 6. Start the Volume

```bash
sudo gluster volume start gv0
```

Check status:

```bash
sudo gluster volume status
```

---

### 7. Mount the Volume to `/var/vmail`

```bash
sudo mkdir -p /var/vmail
sudo mount -t glusterfs server1:/gv0 /var/vmail
```

---

### 8. Make Mount Persistent

Add to `/etc/fstab` on **both servers**:

```
server1:/gv0 /var/vmail glusterfs defaults,_netdev 0 0
```

---

### 9. Set Ownership for iRedMail

After mounting:

```bash
sudo chown -R vmail:vmail /var/vmail
```

---

## Optional: Verify Replication Works

```bash
# On server1
echo "testmail" > /var/vmail/test.txt

# On server2
cat /var/vmail/test.txt
# Output should be: testmail
```

---

## Notes

* Don’t use `/var/vmail` directly as a brick path.
* Never mount a Gluster volume inside its own brick path.
* Gluster replicates all data — ensure sufficient disk space on **both servers**.
