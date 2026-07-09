# RHEL 8 — PCS Active/Passive Cluster for Two Database Servers

**Author:** Sajal Jana  

> **Cluster Type:** Active / Passive  
> **Hardware:** 2× IBM Bare Metal Servers | Dual NIC (`eth0` + `eth1`) | Common SAN Storage  
> **RHEL Version:** RHEL 8.x — PCS 0.10.x / Pacemaker 2.x / Corosync 3.x (knet)  
> **Last Updated:** 2026-03  

---

## Table of Contents

1. [Environment and IP Plan](#1-environment-and-ip-plan)
2. [Architecture Overview](#2-architecture-overview)
3. [Step 1 — NIC Configuration (eth0 Primary + eth1 Heartbeat)](#step-1--nic-configuration-eth0-primary--eth1-heartbeat)
4. [Step 2 — Hostname and /etc/hosts](#step-2--hostname-and-etchosts)
5. [Step 3 — OS Preparation and Subscription](#step-3--os-preparation-and-subscription)
6. [Step 4 — Time Sync with Chrony](#step-4--time-sync-with-chrony)
7. [Step 5 — SAN Multipath Configuration](#step-5--san-multipath-configuration)
8. [Step 6 — LVM on Shared SAN](#step-6--lvm-on-shared-san)
9. [Step 7 — Install Cluster Packages](#step-7--install-cluster-packages)
10. [Step 8 — Configure Firewall](#step-8--configure-firewall)
11. [Step 9 — Set hacluster Password](#step-9--set-hacluster-password)
12. [Step 10 — Authenticate Nodes and Bootstrap Cluster](#step-10--authenticate-nodes-and-bootstrap-cluster)
13. [Step 11 — Configure Corosync Dual-Link (knet)](#step-11--configure-corosync-dual-link-knet)
14. [Step 12 — Configure STONITH via IBM IMM](#step-12--configure-stonith--ibm-imm-via-ipmi)
15. [Step 13 — Create Cluster Resources](#step-13--create-cluster-resources)
16. [Step 14 — Verify and Test Failover](#step-14--verify-and-test-failover)
17. [Cluster Admin Cheat Sheet](#cluster-admin-cheat-sheet)
18. [Troubleshooting](#troubleshooting)

---

## 1. Environment and IP Plan

### Node Details

| Role | Hostname | eth0 — Primary / Client | eth1 — Heartbeat / Corosync |
|------|----------|-------------------------|-----------------------------|
| Node 1 — Active | `db-node1` | `10.240.70.10/24` | `10.240.71.10/24` |
| Node 2 — Passive | `db-node2` | `10.240.70.11/24` | `10.240.71.11/24` |
| Cluster VIP (floats) | `db-cluster` | `10.240.70.20/24` | — |
| Default Gateway | — | `10.240.70.1` | — |

### IBM IMM (Integrated Management Module)

| Node | IBM IMM IP |
|------|------------|
| db-node1 IMM | `10.248.30.10` |
| db-node2 IMM | `10.248.30.11` |

### Storage

| Item | Value |
|------|-------|
| Multipath device | `/dev/mapper/mpatha` |
| Volume Group | `db_vg` |
| Logical Volume | `db_lv` |
| Mount Point | `/data/db` |
| Filesystem | `XFS` |

---

## 2. Architecture Overview
```
                   +------------------------------------------+
  DB Clients ----> |   Virtual IP (VIP): 10.240.70.20         |
                   +-------------------+----------------------+
                                       | Active node holds VIP
              +------------------------+------------------------+
              |                                                 |
   +----------+-----------+               +-----------+--------+------+
   |  db-node1  [ACTIVE]  |               |  db-node2  [PASSIVE]     |
   |  eth0: 10.240.70.10  |               |  eth0: 10.240.70.11      |
   |  eth1: 10.240.71.10  |<--corosync--> |  eth1: 10.240.71.11      |
   |                      |  (heartbeat)  |                          |
   |  IMM: 10.248.30.10   |               |  IMM: 10.248.30.11       |
   +----------+-----------+               +----------+---------------+
              |                                      |
              +-------------------+------------------+
                                  | Shared SAN (FC / iSCSI)
                         +--------+-----------+
                         | /dev/mapper/mpatha  |
                         | VG: db_vg           |
                         | LV: db_lv  (XFS)    |
                         | Mount: /data/db      |
                         +--------------------+

  STONITH (IBM IMM):
    db-node2 fences db-node1 via IMM 10.248.30.10
    db-node1 fences db-node2 via IMM 10.248.30.11
```

**Active/Passive behaviour:**

- All DB resources (LVM, filesystem, VIP, DB service) run on **one node only** at any time.
- If the active node fails, Pacemaker sends a fence command to the IBM IMM of that node, confirms it is powered off, then starts all resources on the passive node.
- Resources are grouped so they always start and stop together in the correct order and never split across nodes.

---

## Step 1 — NIC Configuration (eth0 Primary + eth1 Heartbeat)

> **What it does:** Assigns static IPs to both NICs via `nmcli` (NetworkManager CLI).
> `eth0` carries client/production traffic and will hold the floating VIP.
> `eth1` carries corosync heartbeat traffic only — keeping these on separate NICs
> means a congested or failed client network cannot trigger a false failover.
> `eth1` must **never** become the default route.

### On db-node1
```bash
# --- eth0: Primary NIC ---
# ipv4.method manual   = static IP (disable DHCP)
# ipv4.gateway         = set ONLY on eth0, never on eth1
# connection.autoconnect yes = persist after reboot

nmcli connection modify eth0 \
  ipv4.method manual \
  ipv4.addresses 10.240.70.10/24 \
  ipv4.gateway 10.240.70.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes

# --- eth1: Heartbeat/Corosync NIC ---
# ipv4.gateway ""         = intentionally blank — no default route via eth1
# ipv4.never-default yes  = NetworkManager never sets eth1 as default route

nmcli connection modify eth1 \
  ipv4.method manual \
  ipv4.addresses 10.240.71.10/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes \
  connection.autoconnect yes

# Apply both interfaces
nmcli connection up eth0
nmcli connection up eth1

# Verify IPs
ip addr show eth0
ip addr show eth1

# Confirm only eth0 is the default route
ip route show
# Expected: default via 10.240.70.1 dev eth0 ...  (no default via eth1)
```

### On db-node2
```bash
nmcli connection modify eth0 \
  ipv4.method manual \
  ipv4.addresses 10.240.70.11/24 \
  ipv4.gateway 10.240.70.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes

nmcli connection modify eth1 \
  ipv4.method manual \
  ipv4.addresses 10.240.71.11/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes \
  connection.autoconnect yes

nmcli connection up eth0
nmcli connection up eth1

ip addr show eth0
ip addr show eth1
ip route show
```

> **Pitfall:** If `nmcli connection modify eth0` returns *"Error: unknown connection"*, the
> NetworkManager profile name differs from the device name. Run `nmcli connection show` to
> list all profiles and use the exact name shown (e.g. `"Wired connection 1"`).

---

## Step 2 — Hostname and /etc/hosts

> **What it does:** Cluster daemons (pcs, corosync, pacemaker) communicate by short hostname.
> `/etc/hosts` ensures resolution works even if DNS is unavailable — this is a hard
> requirement for cluster stability.
```bash
# On db-node1:
hostnamectl set-hostname db-node1
hostname   # verify: db-node1

# On db-node2:
hostnamectl set-hostname db-node2
hostname   # verify: db-node2
```

**On BOTH nodes — append cluster entries to /etc/hosts:**
```bash
cat >> /etc/hosts << 'EOF'

# Cluster — Primary (eth0)
10.240.70.10  db-node1
10.240.70.11  db-node2

# Cluster — Heartbeat (eth1)
10.240.71.10  db-node1-hb
10.240.71.11  db-node2-hb

# Cluster VIP
10.240.70.20  db-cluster

# IBM IMM Management
10.248.30.10  db-node1-imm
10.248.30.11  db-node2-imm
EOF

# Verify resolution
ping -c 2 db-node1
ping -c 2 db-node2
ping -c 2 db-node1-hb
ping -c 2 db-node2-hb
```

---

## Step 3 — OS Preparation and Subscription

> **What it does:** Registers the system, attaches a subscription that includes the
> High Availability Add-On, enables the HA and Resilient Storage repos, and performs
> a full package update. Required on both nodes before installing cluster packages.
```bash
# Run on BOTH nodes

# Register with Red Hat CDN
subscription-manager register --username <RH_USER> --password <RH_PASS>

# Auto-attach the best available subscription
subscription-manager attach --auto

# Confirm the HA repo is available
subscription-manager repos --list | grep -i high-availability

# Enable HA and Resilient Storage repos
subscription-manager repos \
  --enable=rhel-8-for-x86_64-highavailability-rpms \
  --enable=rhel-8-for-x86_64-resilientstorage-rpms

# Full package update
dnf update -y

# Verify SELinux is enforcing (cluster packages are SELinux-aware)
getenforce
# If output is Permissive or Disabled:
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
setenforce 1
```

---

## Step 4 — Time Sync with Chrony

> **What it does:** Corosync uses timestamps in its messaging protocol. A clock skew
> greater than ~1 second between nodes can cause authentication errors or spurious
> STONITH events. Chrony is the standard NTP client on RHEL 8 and must be running
> and synced on all cluster nodes.
```bash
# Run on BOTH nodes

dnf install -y chrony
systemctl enable --now chronyd

# Force an immediate clock adjustment
chronyc makestep

# Verify — look for "System clock synchronized: yes"
timedatectl status

# Check active source — the line starting with * is the active time source
chronyc sources -v
```

---

## Step 5 — SAN Multipath Configuration

> **What it does:** `device-mapper-multipath` combines multiple physical FC/iSCSI paths
> to the SAN LUN into a single logical device (`/dev/mapper/mpatha`). If one HBA or
> SAN switch fails, I/O continues on the surviving path with no interruption.
> IBM Storwize / FlashSystem arrays use product string `2145`.
> IBM DS8000 series uses `2107`. Adjust the `product` field for your array model.

### 5a — Install and Enable multipath
```bash
# Run on BOTH nodes

dnf install -y device-mapper-multipath

# mpathconf generates /etc/multipath.conf and enables the service
mpathconf --enable --with_multipathd y

systemctl enable --now multipathd
systemctl status multipathd
# Expected: Active: active (running)
```

### 5b — Configure /etc/multipath.conf for IBM Storage
```bash
# Run on BOTH nodes — config must be IDENTICAL on all nodes

cat > /etc/multipath.conf << 'EOF'
defaults {
    user_friendly_names    yes
    find_multipaths        yes
    path_grouping_policy   multibus
    path_selector          "round-robin 0"
    failback               immediate
    rr_weight              priorities
    no_path_retry          fail
}

blacklist {
    # Prevent multipath from managing the local OS disk
    devnode "^sda$"
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
}

devices {
    # IBM Storwize V-Series / FlashSystem 5000/7000/9000
    # Change product to "2107" for IBM DS8000 series
    device {
        vendor                  "IBM"
        product                 "2145"
        path_grouping_policy    multibus
        path_checker            readsector0
        features                "1 queue_if_no_path"
        hardware_handler        "0"
        prio                    const
        rr_min_io               100
        failback                immediate
    }
}
EOF

# Restart multipathd to apply new config
systemctl restart multipathd

# Rescan all SCSI hosts to discover the SAN LUN
for host in /sys/class/scsi_host/host*; do echo "- - -" > ${host}/scan; done
sleep 3

# Show multipath topology
multipath -ll

# Confirm device node exists
ls -lh /dev/mapper/mpatha
```

**Expected `multipath -ll` output:**
```
mpatha (36005076802810c5f80000000000001a) dm-0 IBM,2145
size=500G features='1 queue_if_no_path' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
   |- 1:0:0:1 sdb 8:16 active ready running
   |- 2:0:0:1 sdc 8:32 active ready running
```

> If `multipath -ll` shows nothing: the LUN has not been zoned/masked to this server.
> Confirm with your SAN admin that both HBA WWPNs are in the fabric zone, then rescan again.

---

## Step 6 — LVM on Shared SAN

> **What it does:** Creates a VG and LV on the shared SAN device. LVM must be configured
> to prevent auto-activation at boot on both nodes simultaneously — that would result in
> dual-active mounts and filesystem corruption. Pacemaker exclusively controls when the
> VG activates via the `LVM-activate` resource agent.

### 6a — Create PV / VG / LV (db-node1 only — first-time setup)
```bash
# Run on db-node1 ONLY

# Create Physical Volume on the multipath device
pvcreate /dev/mapper/mpatha

# Create Volume Group
vgcreate db_vg /dev/mapper/mpatha

# Create Logical Volume — adjust size as required
lvcreate -L 500G -n db_lv db_vg

# Create XFS filesystem
mkfs.xfs /dev/db_vg/db_lv

# Confirm
lvs -o lv_name,lv_size,vg_name
# Expected: db_lv   500.00g   db_vg
```

### 6b — Prevent Auto-Activation at Boot (BOTH nodes)

> This step is critical. Without it both nodes may activate the VG during boot
> and mount the filesystem simultaneously, causing data corruption.
```bash
# Run on BOTH nodes

# Back up lvm.conf
cp /etc/lvm/lvm.conf /etc/lvm/lvm.conf.bak

# Check your OS VG name (typically 'rhel' on RHEL 8)
vgs

# Restrict LVM boot-time activation to the OS VG only.
# db_vg is deliberately excluded — Pacemaker activates it exclusively.
# If your OS VG has a different name, replace "rhel" accordingly.

sed -i '/# auto_activation_volume_list/a\    auto_activation_volume_list = [ "rhel" ]' \
  /etc/lvm/lvm.conf

# Rebuild initramfs so the change takes effect at next boot
dracut -H -f
```

### 6c — Create Mount Point (BOTH nodes)
```bash
mkdir -p /data/db
# Do NOT add /data/db to /etc/fstab — Pacemaker manages mounting
```

---

## Step 7 — Install Cluster Packages

> **What it does:** Installs the three-layer cluster stack:
>
> - `corosync` — messaging and membership layer (heartbeat between nodes)
> - `pacemaker` — resource manager (decides where resources run, handles failover)
> - `pcs` — CLI that wraps both pacemaker and corosync; the primary admin tool
> - `fence-agents-all` — STONITH agent collection; includes `fence_ipmilan` for IBM IMM
> - `pcsd` — the PCS daemon listening on TCP 2224; required for `pcs auth`
```bash
# Run on BOTH nodes

dnf install -y pacemaker pcs fence-agents-all

# Enable and start pcsd — must be running BEFORE pcs auth
systemctl enable --now pcsd

systemctl status pcsd
# Expected: Active: active (running)
```

---

## Step 8 — Configure Firewall

> **What it does:** Opens all cluster-required ports using the predefined
> `high-availability` firewalld service group. This opens:
>
> - TCP 2224 — pcsd API (pcs auth and cluster management)
> - TCP 3121 — pacemaker_remote
> - UDP 5404 — corosync multicast
> - UDP 5405 — corosync unicast (knet transport)
> - TCP 21064 — DLM (Distributed Lock Manager)
```bash
# Run on BOTH nodes

firewall-cmd --permanent --add-service=high-availability

# If eth1 (heartbeat) is in a different firewalld zone, add to that zone too.
# First check which zone each NIC is in:
firewall-cmd --get-active-zones

# Example — if eth1 is in the 'internal' zone:
# firewall-cmd --permanent --zone=internal --add-service=high-availability

# Apply rules
firewall-cmd --reload

# Verify
firewall-cmd --list-services
# Expected output includes: high-availability
```

---

## Step 9 — Set hacluster Password

> **What it does:** The `hacluster` OS user is created automatically by the pacemaker
> package. `pcs host auth` uses this account's password to exchange session tokens
> between pcsd daemons. The **same password must be set on all nodes** — authentication
> will fail if they differ.
```bash
# Run on BOTH nodes — use the SAME password on both

echo "ClusterP@ss2024!" | passwd --stdin hacluster

# Verify the account exists
id hacluster
# Expected: uid=189(hacluster) gid=189(hacluster) groups=189(hacluster),982(haclient)
```

---

## Step 10 — Authenticate Nodes and Bootstrap Cluster

> **What it does:** `pcs host auth` exchanges authentication tokens between the pcsd
> daemons on each node (stored in `/var/lib/pcsd/`). After this, pcs commands can
> target either node without re-entering credentials. `pcs cluster setup` generates
> `corosync.conf` and pushes it to all nodes. These commands run from **db-node1 only**.
```bash
# Run on db-node1 ONLY

# Step 10a — Authenticate both nodes
# -u hacluster : the OS account used for authentication
# -p           : the password set in Step 9

pcs host auth db-node1 db-node2 \
  -u hacluster \
  -p 'ClusterP@ss2024!'

# Expected output:
# db-node1: Authorized
# db-node2: Authorized

# Step 10b — Create the cluster
# This generates /etc/corosync/corosync.conf with a single-link config (eth0 only).
# We extend it to dual-link in Step 11.
# --force clears any leftover cluster state from previous attempts.

pcs cluster setup db-cluster db-node1 db-node2 --force

# Step 10c — Start cluster on all nodes
pcs cluster start --all

# Allow ~20s for Pacemaker to elect a DC (Designated Coordinator)
sleep 20
pcs status

# Step 10d — Enable cluster to start automatically after reboot
pcs cluster enable --all
```

---

## Step 11 — Configure Corosync Dual-Link (knet)

> **What it does:** By default `pcs cluster setup` configures corosync using only eth0
> (link 0). Adding link 1 on eth1 gives corosync **two independent communication paths**.
> If eth0 becomes congested or fails completely, corosync continues heartbeating over eth1
> and no false failover occurs. RHEL 8 uses corosync 3.x with `knet` transport, which
> supports multiple links natively — no transport change is needed.
```bash
# Step 11a — Stop the cluster before editing corosync.conf
pcs cluster stop --all

# Step 11b — Back up the auto-generated config
cp /etc/corosync/corosync.conf /etc/corosync/corosync.conf.bak

# Step 11c — Write the dual-link corosync.conf
# This file must be IDENTICAL on both nodes

cat > /etc/corosync/corosync.conf << 'EOF'
# corosync.conf — db-cluster
# knet transport with two links:
#   link 0 = eth0  primary     (10.240.70.x)
#   link 1 = eth1  heartbeat   (10.240.71.x)

totem {
    version:                2
    cluster_name:           db-cluster
    transport:              knet

    interface {
        linknumber:         0
        knet_link_priority: 1
    }

    interface {
        linknumber:         1
        knet_link_priority: 2
    }

    crypto_hash:            sha256
    crypto_cipher:          aes256
}

nodelist {
    node {
        name:         db-node1
        nodeid:       1
        ring0_addr:   10.240.70.10
        ring1_addr:   10.240.71.10
    }
    node {
        name:         db-node2
        nodeid:       2
        ring0_addr:   10.240.70.11
        ring1_addr:   10.240.71.11
    }
}

quorum {
    provider:       corosync_votequorum
    # two_node: 1 allows a single surviving node to achieve quorum
    # ONLY after STONITH has confirmed the other node is powered off
    two_node:       1
}

logging {
    to_logfile:     yes
    logfile:        /var/log/cluster/corosync.log
    to_syslog:      yes
    timestamp:      on
}
EOF

# Step 11d — Push identical config to db-node2
scp /etc/corosync/corosync.conf root@db-node2:/etc/corosync/corosync.conf

# Step 11e — Start the cluster
pcs cluster start --all
sleep 20

# Step 11f — Verify both corosync links are active
corosync-cfgtool -s

# Expected output:
# Local node ID 1
# RING ID 0
#   id     = 10.240.70.10
#   status = ring 0 active with no faults
# RING ID 1
#   id     = 10.240.71.10
#   status = ring 1 active with no faults
```

---

## Step 12 — Configure STONITH — IBM IMM via IPMI

> **What it does:** STONITH (Shoot The Other Node In The Head) is **mandatory** in
> production clusters. Without it Pacemaker will not start resources after a node
> failure because it cannot confirm the failed node is not still writing to shared
> storage. IBM IMM (Integrated Management Module) exposes IPMI over LAN;
> `fence_ipmilan` sends a hard power-off command directly to the server's baseboard
> management controller — bypassing the OS entirely.
>
> **Best practice:** Each node's fence device runs preferentially on the opposite node.
> If db-node1 hangs, db-node2 is already running the agent that kills it.

### 12a — Verify IPMI Connectivity to IBM IMM
```bash
# From db-node1 — test reach to db-node2's IMM
# -o status  = query power state only; does NOT change power state (safe test)
# --lanplus  = required for IPMI v2.0 (all modern IBM IMM)

fence_ipmilan \
  -a 10.248.30.11 \
  -l ADMIN \
  -p 'IMMpassword!' \
  --lanplus \
  -o status
# Expected: Status: ON

# From db-node2 — test reach to db-node1's IMM
fence_ipmilan \
  -a 10.248.30.10 \
  -l ADMIN \
  -p 'IMMpassword!' \
  --lanplus \
  -o status
# Expected: Status: ON

# If unreachable:
#   ping 10.248.30.10 / 10.248.30.11
#   In IBM IMM web UI: Settings > Network > IPMI over LAN = Enabled
```

### 12b — Create STONITH Resources
```bash
# Run on db-node1

# Fence resource for db-node1 — controlled from db-node2's side
# ipaddr         = IBM IMM IP of the node being fenced
# pcmk_host_list = hostname this fence device targets
# power_wait     = seconds to wait after power action before checking result

pcs stonith create fence-db-node1 fence_ipmilan \
  ipaddr="10.248.30.10" \
  login="ADMIN" \
  passwd="IMMpassword!" \
  lanplus="1" \
  pcmk_host_list="db-node1" \
  power_wait="4" \
  op monitor interval=60s

# Fence resource for db-node2 — controlled from db-node1's side
pcs stonith create fence-db-node2 fence_ipmilan \
  ipaddr="10.248.30.11" \
  login="ADMIN" \
  passwd="IMMpassword!" \
  lanplus="1" \
  pcmk_host_list="db-node2" \
  power_wait="4" \
  op monitor interval=60s

# Location constraints — each fence device prefers to run on the OPPOSITE node
pcs constraint location fence-db-node1 prefers db-node2=INFINITY
pcs constraint location fence-db-node2 prefers db-node1=INFINITY

# Verify STONITH resources are started
pcs stonith show

# Expected:
#  fence-db-node1  (stonith:fence_ipmilan):  Started db-node2
#  fence-db-node2  (stonith:fence_ipmilan):  Started db-node1
```

---

## Step 13 — Create Cluster Resources

> **What it does:** Defines four Pacemaker-managed resources and groups them so
> they always run together on one node in the correct order.
>
> **Start order:** `db_lvm → db_fs → db_vip → db_service`
> **Stop order (automatic reverse):** `db_service → db_vip → db_fs → db_lvm`

### 13a — Disable DB Service in systemd (BOTH nodes)

> Pacemaker must be the sole controller of the DB service. If systemd starts it
> independently at boot, Pacemaker loses track and may start a second instance.
```bash
# Run on BOTH nodes

# PostgreSQL:
systemctl disable postgresql
systemctl stop postgresql

# MariaDB (if using MariaDB instead):
# systemctl disable mariadb
# systemctl stop mariadb
```

### 13b — Virtual IP Resource
```bash
# IPaddr2 RA adds/removes the VIP on the active node's eth0
# The VIP 10.240.70.20 floats to whichever node is active

pcs resource create db_vip IPaddr2 \
  ip=10.240.70.20 \
  cidr_netmask=24 \
  nic=eth0 \
  op monitor interval=30s
```

### 13c — LVM Activation Resource
```bash
# LVM-activate RA activates/deactivates the shared VG on the active node
# vg_access_mode=system_id prevents both nodes activating the VG simultaneously

pcs resource create db_lvm LVM-activate \
  vgname=db_vg \
  vg_access_mode=system_id \
  activation_mode=exclusive \
  op monitor interval=30s
```

### 13d — Filesystem Resource
```bash
# Filesystem RA handles mount, unmount, and fsck of the LV

pcs resource create db_fs Filesystem \
  device="/dev/db_vg/db_lv" \
  directory="/data/db" \
  fstype="xfs" \
  op monitor interval=30s \
  op start timeout=60s \
  op stop timeout=60s
```

### 13e — Database Service Resource
```bash
# systemd RA wraps a systemd unit — Pacemaker starts, stops, and monitors it

# For PostgreSQL:
pcs resource create db_service systemd:postgresql \
  op monitor interval=30s \
  op start timeout=120s \
  op stop timeout=120s

# For MariaDB, replace systemd:postgresql with systemd:mariadb
```

### 13f — Group All Resources
```bash
# Grouping enforces:
#   - All resources run on the SAME node
#   - Start in left-to-right order
#   - Stop in right-to-left order

pcs resource group add db-group \
  db_lvm \
  db_fs \
  db_vip \
  db_service

# Prevent automatic failback — resource stays on new node after failover
# (avoids unnecessary disruption if original node recovers)
pcs resource defaults update resource-stickiness=100

# After 3 consecutive failures on a node, migrate to the other node
pcs resource defaults update migration-threshold=3

# Final status check
pcs status
```

**Expected `pcs status` output:**
```
Cluster name: db-cluster
Stack: corosync
Current DC: db-node1 - partition with quorum
2 nodes configured
6 resource instances configured

Online: [ db-node1 db-node2 ]

Full list of resources:

 fence-db-node1  (stonith:fence_ipmilan):   Started db-node2
 fence-db-node2  (stonith:fence_ipmilan):   Started db-node1
 Resource Group: db-group
     db_lvm     (ocf::heartbeat:LVM-activate):  Started db-node1
     db_fs      (ocf::heartbeat:Filesystem):     Started db-node1
     db_vip     (ocf::heartbeat:IPaddr2):        Started db-node1
     db_service (systemd:postgresql):            Started db-node1
```

---

## Step 14 — Verify and Test Failover

### 14a — Corosync and Quorum Check
```bash
# Both links must show: active with no faults
corosync-cfgtool -s

# Quorum must be achieved
pcs quorum status
# Expected: Quorum provided

# Full cluster config check
pcs status --full
```

### 14b — Manual Failover via Standby (Safe — No Power Cycle)
```bash
# standby evacuates db-node1 without powering it off
# The db-group migrates to db-node2

pcs node standby db-node1

# Watch failover in real time (~15-30 seconds)
watch pcs status

# Confirm VIP moved — run on db-node2:
ip addr show eth0 | grep 10.240.70.20
# Expected: inet 10.240.70.20/24 ...

# Bring db-node1 back
pcs node unstandby db-node1

# Resources stay on db-node2 due to stickiness=100
# To move them back manually:
pcs resource move db-group db-node1
pcs resource clear db-group   # remove temporary constraint after confirming
```

### 14c — STONITH Fence Test (DESTRUCTIVE — Maintenance Window Only)
```bash
# WARNING: This immediately powers off db-node2 via IBM IMM

pcs stonith fence db-node2

# Pacemaker confirms db-node2 is off, then starts resources on db-node1
watch pcs status

# Power db-node2 back on via IMM web UI or physical button
# Once booted and pcsd is running, it rejoins automatically

pcs status
# Expected: both nodes Online
```

### 14d — Verify DB Connectivity via VIP
```bash
# From any host on 10.240.70.0/24
ping -c 4 10.240.70.20

# PostgreSQL:
psql -h 10.240.70.20 -U postgres -c "SELECT version();"

# MariaDB:
mysql -h 10.240.70.20 -u root -p -e "SELECT VERSION();"
```

---

## Cluster Admin Cheat Sheet

| Task | Command |
|------|---------|
| Full cluster status | `pcs status` |
| Resource status | `pcs resource show` |
| STONITH status | `pcs stonith show` |
| All constraints | `pcs constraint show` |
| Start cluster — all nodes | `pcs cluster start --all` |
| Stop cluster — all nodes | `pcs cluster stop --all` |
| Put node in standby | `pcs node standby <node>` |
| Remove node from standby | `pcs node unstandby <node>` |
| Move resource group | `pcs resource move db-group <node>` |
| Clear move constraint | `pcs resource clear db-group` |
| Disable a resource | `pcs resource disable <resource>` |
| Enable a resource | `pcs resource enable <resource>` |
| Clean up failed resource | `pcs resource cleanup <resource>` |
| View failure counts | `pcs resource failcount show` |
| Corosync link status | `corosync-cfgtool -s` |
| Quorum status | `pcs quorum status` |
| Live cluster logs | `journalctl -u corosync -u pacemaker -f` |
| Corosync log file | `tail -f /var/log/cluster/corosync.log` |
| Dump cluster CIB (config) | `pcs cluster cib` |
| Save CIB to file | `pcs cluster cib > /tmp/cluster-cib.xml` |
| Manually fence a node | `pcs stonith fence <node>` |
| Fencing history | `pcs stonith history` |

---

## Troubleshooting

### Node shows OFFLINE or UNCLEAN
```bash
# Check all three cluster daemons on the affected node
systemctl status pcsd corosync pacemaker

# Detailed corosync logs
journalctl -u corosync --no-pager -n 50

# Verify firewall is not blocking cluster ports
firewall-cmd --list-services   # must include: high-availability

# Verify hostname resolution — cluster tools rely on this
ping db-node1
ping db-node2
ping db-node1-hb
ping db-node2-hb
```

### Resource stuck in FAILED or keeps restarting
```bash
# View failure reason in detail
pcs resource failcount show
pcs status --full | grep -A 5 FAILED

# Check pacemaker logs for resource agent error output
journalctl -u pacemaker --no-pager -n 100 | grep -iE "error|fail|warning"

# Clear failure history and allow Pacemaker to retry
pcs resource cleanup db-group
```

### Corosync ring shows FAULTY
```bash
corosync-cfgtool -s

# If ring 1 (eth1) shows FAULTY, check:
ip link show eth1                   # must be UP
ip addr show eth1                   # must show 10.240.71.x
ip route show | grep 10.240.71      # route must exist
# Also inspect physical switch port and patch cable for eth1
```

### STONITH fails — resources will not start after node failure
```bash
# Pacemaker will not start resources unless it is certain the failed node
# is powered off. If auto-fencing fails, physically confirm the node is
# off then manually confirm to Pacemaker:

pcs stonith confirm db-node2

# Re-test fence agent connectivity
fence_ipmilan \
  -a 10.248.30.11 \
  -l ADMIN \
  -p 'IMMpassword!' \
  --lanplus \
  -o status
```

### LVM fails to activate on failover
```bash
# Check for stale system_id from the previous active node
vgs -o+systemid db_vg

# If a stale system_id is shown, clear it:
vgchange --systemid "" db_vg

# Verify LVM-activate resource configuration
pcs resource show db_lvm
# pcs resource move webservice node2 (in cluster suite #clusvcadm -r service -m node2) 
```

### Two-node quorum — resources will not start after one node disappears
```bash
# This is correct and expected behaviour.
# One node cannot achieve quorum alone unless STONITH confirms the other is off.
# NEVER set no-quorum-policy=ignore when shared storage is involved.
# The solution is to ensure IBM IMM connectivity and STONITH are working.

# Verify quorum policy is correct:
pcs property show no-quorum-policy
# Expected: no-quorum-policy: stop
```

---

> **References:**
>
> - [RHEL 8 — Configuring and Managing High Availability Clusters](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_high_availability_clusters/)
> - [Pacemaker Resource Agents Reference](https://github.com/ClusterLabs/resource-agents)
> - `man corosync.conf` | `man pcs` | `man fence_ipmilan`
> - IBM Documentation: *Enabling IPMI over LAN on IBM System x and BladeCenter*
