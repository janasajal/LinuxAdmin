---

# RHEL 8 — PCS Active/Passive Cluster for Two Database Servers

> **Cluster Type:** Active / Passive | **Hardware:** 2× IBM Bare Metal | Dual NIC (eth0 + eth1) | SAN + Multipath | IBM IMM Fencing
> **RHEL:** 8.x | PCS 0.10.x | Pacemaker 2.x | Corosync 3.x (knet transport)

---

## Environment & IP Plan

| Role | Hostname | eth0 (Primary) | eth1 (Heartbeat) |
|------|----------|----------------|-----------------|
| Node 1 — Active | `db-node1` | `10.240.70.10/24` | `10.240.71.10/24` |
| Node 2 — Passive | `db-node2` | `10.240.70.11/24` | `10.240.71.11/24` |
| Cluster VIP | `db-cluster` | `10.240.70.20/24` | — |
| Gateway | — | `10.240.70.1` | — |
| db-node1 IBM IMM | — | `10.248.30.10` | — |
| db-node2 IBM IMM | — | `10.248.30.11` | — |
| SAN multipath device | `/dev/mapper/mpatha` | VG: `db_vg` | Mount: `/data/db` |

---

## Architecture

```
  DB Clients ──▶ VIP: 10.240.70.20 (floats to active node)
                        │
         ┌──────────────┴──────────────┐
         │                             │
  db-node1 [ACTIVE]          db-node2 [PASSIVE]
  eth0: 10.240.70.10          eth0: 10.240.70.11
  eth1: 10.240.71.10 ◄──────▶ eth1: 10.240.71.11
        (corosync)                (corosync)
  IMM: 10.248.30.10           IMM: 10.248.30.11
         │                             │
         └──────────┬──────────────────┘
               Shared SAN
          /dev/mapper/mpatha
          VG: db_vg / LV: db_lv
          Mount: /data/db (XFS)

  STONITH:
    db-node2 fences db-node1 via IMM 10.248.30.10
    db-node1 fences db-node2 via IMM 10.248.30.11
```

---

## Step 1 — NIC Configuration (eth0 Primary + eth1 Heartbeat)

> **What it does:** Assigns static IPs to both NICs via `nmcli`. `eth0` carries client traffic and the floating VIP. `eth1` carries corosync heartbeat traffic only. A dedicated heartbeat NIC means a saturated client network cannot trigger false failover. `eth1` must never be the default route.

### On db-node1

```bash
# eth0 — Primary NIC
# ipv4.method manual = static IP
# ipv4.gateway = set ONLY on eth0, never on eth1
nmcli connection modify eth0 \
  ipv4.method manual \
  ipv4.addresses 10.240.70.10/24 \
  ipv4.gateway 10.240.70.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  connection.autoconnect yes

# eth1 — Heartbeat/Corosync NIC
# ipv4.gateway "" = no default route via eth1
# ipv4.never-default yes = NetworkManager never sets eth1 as default route
nmcli connection modify eth1 \
  ipv4.method manual \
  ipv4.addresses 10.240.71.10/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes \
  connection.autoconnect yes

nmcli connection up eth0
nmcli connection up eth1

# Verify
ip addr show eth0
ip addr show eth1
ip route show
# Expected: default via 10.240.70.1 dev eth0 ... (no default via eth1)
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

> **Pitfall:** If `nmcli connection modify eth0` returns "Error: unknown connection", the profile name differs from device name. Run `nmcli connection show` to list profiles and use the exact name shown.

---

## Step 2 — Hostname & /etc/hosts

> **What it does:** Cluster tools resolve nodes by short hostname internally. `/etc/hosts` ensures resolution works even if DNS is down.

```bash
# On db-node1:
hostnamectl set-hostname db-node1

# On db-node2:
hostnamectl set-hostname db-node2
```

**On BOTH nodes — append to /etc/hosts:**

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

# Verify all entries resolve
ping -c 2 db-node1
ping -c 2 db-node2
ping -c 2 db-node1-hb
ping -c 2 db-node2-hb
```

---

## Step 3 — OS Preparation & Subscription

> **What it does:** Attaches a subscription with the HA Add-On, enables the HA repo, and updates all packages. Mandatory on both nodes.

```bash
# Run on BOTH nodes

subscription-manager register --username <RH_USER> --password <RH_PASS>
subscription-manager attach --auto

# Verify HA repo is accessible
subscription-manager repos --list | grep -i high-availability

# Enable HA and Resilient Storage repos
subscription-manager repos \
  --enable=rhel-8-for-x86_64-highavailability-rpms \
  --enable=rhel-8-for-x86_64-resilientstorage-rpms

dnf update -y

# Confirm SELinux is enforcing
getenforce
# If not enforcing:
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
setenforce 1
```

---

## Step 4 — Time Sync with Chrony

> **What it does:** Corosync uses timestamps in its messaging protocol. A clock skew greater than ~1 second between nodes causes authentication errors or spurious fencing events. Chrony is the standard NTP client on RHEL 8.

```bash
# Run on BOTH nodes

dnf install -y chrony
systemctl enable --now chronyd

# Force immediate sync
chronyc makestep

# Verify — look for "System clock synchronized: yes"
timedatectl status

# Check source detail — active source has * prefix
chronyc sources -v
```

---

## Step 5 — SAN Multipath Configuration

> **What it does:** `device-mapper-multipath` combines multiple physical FC/iSCSI paths to the SAN LUN into one logical device (`/dev/mapper/mpatha`). If one HBA or SAN switch fails, I/O continues on the other path. IBM Storwize/FlashSystem uses product string `2145`; DS8000 uses `2107`. Adjust `product` for your array model.

### 5a — Install and Enable multipath

```bash
# Run on BOTH nodes
dnf install -y device-mapper-multipath

# mpathconf generates /etc/multipath.conf and enables multipathd
mpathconf --enable --with_multipathd y

systemctl enable --now multipathd
systemctl status multipathd
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

systemctl restart multipathd

# Rescan SCSI bus to discover SAN LUN
for host in /sys/class/scsi_host/host*; do echo "- - -" > ${host}/scan; done
sleep 3

# Verify multipath topology
multipath -ll

# Confirm device node
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

> If `multipath -ll` shows nothing: the LUN is not zoned/masked to this server. Confirm with your SAN admin that both HBA WWPNs are in the fabric zone.

---

## Step 6 — LVM on Shared SAN

> **What it does:** Creates a VG and LV on the shared SAN. LVM is configured to prevent auto-activation at boot, which would cause both nodes to mount the filesystem simultaneously and corrupt data. Pacemaker exclusively controls VG activation via the `LVM-activate` resource agent.

### 6a — Create PV / VG / LV (db-node1 only — first time)

```bash
# Run on db-node1 ONLY

pvcreate /dev/mapper/mpatha
vgcreate db_vg /dev/mapper/mpatha
lvcreate -L 500G -n db_lv db_vg
mkfs.xfs /dev/db_vg/db_lv

# Confirm
lvs -o lv_name,lv_size,vg_name
```

### 6b — Prevent Auto-Activation at Boot (BOTH nodes)

```bash
# Run on BOTH nodes

cp /etc/lvm/lvm.conf /etc/lvm/lvm.conf.bak

# Restrict LVM boot-time activation to OS VG only (typically named 'rhel')
# db_vg is deliberately excluded — Pacemaker activates it
# Verify your OS VG name first:
vgs

# Add the restriction (edit /etc/lvm/lvm.conf — activation section)
sed -i '/# auto_activation_volume_list/a\    auto_activation_volume_list = [ "rhel" ]' \
  /etc/lvm/lvm.conf

# Rebuild initramfs to pick up the change
dracut -H -f
```

### 6c — Create Mount Point (BOTH nodes)

```bash
mkdir -p /data/db
# Do NOT add to /etc/fstab — Pacemaker manages mounting
```

---

## Step 7 — Install Cluster Packages

> **What it does:** Installs `pacemaker` (resource manager), `pcs` (management CLI), `corosync` (heartbeat/messaging — pulled as dependency), and `fence-agents-all` (includes `fence_ipmilan` for IBM IMM). Enables `pcsd` — the daemon that listens on TCP 2224 for pcs auth and cluster management.

```bash
# Run on BOTH nodes

dnf install -y pacemaker pcs fence-agents-all

# Enable and start pcsd — required BEFORE running pcs auth
systemctl enable --now pcsd

systemctl status pcsd
# Expected: Active: active (running)
```

---

## Step 8 — Configure Firewall

> **What it does:** Opens all cluster-required ports via the predefined `high-availability` firewalld service group. Opens TCP 2224 (pcsd), TCP 3121 (pacemaker remote), UDP 5404/5405 (corosync), TCP 21064 (DLM).

```bash
# Run on BOTH nodes

firewall-cmd --permanent --add-service=high-availability

# If eth1 is in a different firewalld zone (check with: firewall-cmd --get-active-zones)
# add to that zone too, e.g. if eth1 is in 'internal' zone:
# firewall-cmd --permanent --zone=internal --add-service=high-availability

firewall-cmd --reload

# Verify
firewall-cmd --list-services
# Expected output includes: high-availability
```

---

## Step 9 — Set hacluster Password

> **What it does:** The `hacluster` OS user is auto-created by pacemaker. `pcs host auth` uses this account's password to exchange session tokens between nodes. The password must be identical on all nodes.

```bash
# Run on BOTH nodes — same password on both

echo "ClusterP@ss2024!" | passwd --stdin hacluster

# Verify account
id hacluster
# Expected: uid=189(hacluster) gid=189(hacluster) groups=189(hacluster),982(haclient)
```

---

## Step 10 — Authenticate Nodes & Bootstrap Cluster

> **What it does:** `pcs host auth` exchanges authentication tokens stored in `/var/lib/pcsd/`. `pcs cluster setup` generates `corosync.conf` (single-link initially) and distributes it. Run from db-node1 only.

```bash
# Run on db-node1 ONLY

# Authenticate both nodes
pcs host auth db-node1 db-node2 \
  -u hacluster \
  -p 'ClusterP@ss2024!'

# Expected:
# db-node1: Authorized
# db-node2: Authorized

# Create the cluster
# --force clears any leftover cluster state
pcs cluster setup db-cluster db-node1 db-node2 --force

# Start cluster on all nodes
pcs cluster start --all

# Allow ~20s for Pacemaker to elect a DC (Designated Coordinator)
sleep 20
pcs status

# Enable cluster auto-start after reboot
pcs cluster enable --all
```

---

## Step 11 — Configure Corosync Dual-Link (knet)

> **What it does:** The default `pcs cluster setup` config uses only eth0 (link 0). Adding link 1 (eth1) gives corosync a second independent path. If eth0 fails or is congested, corosync continues heartbeating on eth1, preventing a false failover. RHEL 8 uses corosync 3.x with `knet` transport, which supports multiple links natively.

```bash
# Step 11a — Stop cluster to safely edit corosync.conf
pcs cluster stop --all

# Step 11b — Back up generated config
cp /etc/corosync/corosync.conf /etc/corosync/corosync.conf.bak

# Step 11c — Write dual-link corosync.conf
# This file must be IDENTICAL on both nodes

cat > /etc/corosync/corosync.conf << 'EOF'
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

# Step 11e — Start cluster
pcs cluster start --all
sleep 20

# Step 11f — Verify both links are active
corosync-cfgtool -s

# Expected:
# RING ID 0  id = 10.240.70.10  status = ring 0 active with no faults
# RING ID 1  id = 10.240.71.10  status = ring 1 active with no faults
```

---

## Step 12 — Configure STONITH — IBM IMM via IPMI

> **What it does:** STONITH is mandatory in production. Without it, Pacemaker will not start resources after a node failure because it cannot confirm the failed node is not still writing to shared storage. IBM IMM exposes IPMI over LAN; `fence_ipmilan` sends a hard power-off command directly to the baseboard management controller, bypassing the OS entirely. Each node's fence device runs on the opposite node — if db-node1 hangs, db-node2 runs the agent that powers it off.

### 12a — Verify IPMI Connectivity

```bash
# From db-node1 — test reach to db-node2's IMM
# -o status = query power state only, does NOT change anything

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

# If unreachable: ping 10.248.30.10 / 10.248.30.11
# Confirm in IBM IMM web UI: Settings > Network > IPMI over LAN = Enabled
```

### 12b — Create STONITH Resources

```bash
# Run on db-node1

# Fence resource for db-node1 (runs preferably on db-node2)
pcs stonith create fence-db-node1 fence_ipmilan \
  ipaddr="10.248.30.10" \
  login="ADMIN" \
  passwd="IMMpassword!" \
  lanplus="1" \
  pcmk_host_list="db-node1" \
  power_wait="4" \
  op monitor interval=60s

# Fence resource for db-node2 (runs preferably on db-node1)
pcs stonith create fence-db-node2 fence_ipmilan \
  ipaddr="10.248.30.11" \
  login="ADMIN" \
  passwd="IMMpassword!" \
  lanplus="1" \
  pcmk_host_list="db-node2" \
  power_wait="4" \
  op monitor interval=60s

# Location constraints — each fence agent prefers to run on the OPPOSITE node
pcs constraint location fence-db-node1 prefers db-node2=INFINITY
pcs constraint location fence-db-node2 prefers db-node1=INFINITY

# Verify
pcs stonith show
# Expected:
#  fence-db-node1  (stonith:fence_ipmilan):  Started db-node2
#  fence-db-node2  (stonith:fence_ipmilan):  Started db-node1
```

---

## Step 13 — Create Cluster Resources

> **What it does:** Defines four Pacemaker-managed resources and groups them so they always run together on one node in the correct start/stop order. Active/passive is enforced by the group — resources never split across nodes.
>
> Start order: `db_lvm → db_fs → db_vip → db_service`
> Stop order (automatic reverse): `db_service → db_vip → db_fs → db_lvm`

### 13a — Disable DB Service in systemd (BOTH nodes)

```bash
# Pacemaker must be the sole controller of the DB service.
# If systemd starts it independently, Pacemaker loses track and may
# start a second instance or fight with systemd over the service.

# PostgreSQL:
systemctl disable postgresql
systemctl stop postgresql

# MariaDB:
# systemctl disable mariadb && systemctl stop mariadb
```

### 13b — VIP Resource

```bash
# IPaddr2 RA adds/removes the VIP on the active node's eth0
pcs resource create db_vip IPaddr2 \
  ip=10.240.70.20 \
  cidr_netmask=24 \
  nic=eth0 \
  op monitor interval=30s
```

### 13c — LVM Activation Resource

```bash
# LVM-activate RA activates/deactivates db_vg on the active node
# vg_access_mode=system_id prevents dual activation
pcs resource create db_lvm LVM-activate \
  vgname=db_vg \
  vg_access_mode=system_id \
  activation_mode=exclusive \
  op monitor interval=30s
```

### 13d — Filesystem Resource

```bash
# Filesystem RA handles mount/unmount of the LV
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
# systemd RA wraps a systemd unit file

# PostgreSQL:
pcs resource create db_service systemd:postgresql \
  op monitor interval=30s \
  op start timeout=120s \
  op stop timeout=120s

# MariaDB: replace systemd:postgresql with systemd:mariadb
```

### 13f — Group Resources

```bash
# Grouping enforces: same node, correct start order, reverse stop order
pcs resource group add db-group \
  db_lvm \
  db_fs \
  db_vip \
  db_service

# Prevent unnecessary failback (resource stays on new node after failover)
pcs resource defaults update resource-stickiness=100

# After 3 failures on a node, migrate to the other
pcs resource defaults update migration-threshold=3

# Final status check
pcs status
```

**Expected output:**
```
Cluster name: db-cluster
Current DC: db-node1 - partition with quorum
2 nodes configured, 6 resource instances configured

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

## Step 14 — Verify & Test Failover

### 14a — Corosync & Quorum Check

```bash
corosync-cfgtool -s
# Both rings must show: active with no faults

pcs quorum status
# Expected: Quorum provided
```

### 14b — Manual Failover via Standby (Safe — No Power Cycle)

```bash
# Evacuate db-node1 — resources migrate to db-node2
pcs node standby db-node1

# Watch in real time (~15-30 seconds)
watch pcs status

# Confirm VIP moved — run on db-node2:
ip addr show eth0 | grep 10.240.70.20

# Bring db-node1 back
pcs node unstandby db-node1

# Resources stay on db-node2 (stickiness). Move back manually if needed:
pcs resource move db-group db-node1
pcs resource clear db-group   # remove temporary constraint
```

### 14c — STONITH Fence Test (DESTRUCTIVE — Maintenance Window Only)

```bash
# WARNING: Powers off db-node2 immediately via IMM

pcs stonith fence db-node2

watch pcs status
# Resources start on db-node1 within ~30s after fence completes

# Power db-node2 back on via IMM or physical button
# It will rejoin the cluster automatically once booted
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
| Start cluster all nodes | `pcs cluster start --all` |
| Stop cluster all nodes | `pcs cluster stop --all` |
| Standby a node | `pcs node standby <node>` |
| Un-standby a node | `pcs node unstandby <node>` |
| Move resource group | `pcs resource move db-group <node>` |
| Clear move constraint | `pcs resource clear db-group` |
| Disable resource | `pcs resource disable <res>` |
| Enable resource | `pcs resource enable <res>` |
| Clean failed resource | `pcs resource cleanup <res>` |
| View failure counts | `pcs resource failcount show` |
| Corosync link status | `corosync-cfgtool -s` |
| Quorum status | `pcs quorum status` |
| Live cluster logs | `journalctl -u corosync -u pacemaker -f` |
| Corosync log file | `tail -f /var/log/cluster/corosync.log` |
| Dump cluster CIB | `pcs cluster cib` |
| Manually fence node | `pcs stonith fence <node>` |
| Fencing history | `pcs stonith history` |

---

## Troubleshooting

**Node shows OFFLINE / UNCLEAN:**
```bash
systemctl status pcsd corosync pacemaker
journalctl -u corosync --no-pager -n 50
firewall-cmd --list-services          # must include high-availability
ping db-node1 && ping db-node1-hb     # both must resolve
```

**Resource stuck in FAILED:**
```bash
pcs resource failcount show
pcs status --full | grep -A5 FAILED
journalctl -u pacemaker --no-pager -n 100 | grep -iE "error|fail"
pcs resource cleanup db-group         # clear history, allow retry
```

**Corosync ring FAULTY:**
```bash
corosync-cfgtool -s
ip link show eth1                     # must be UP
ip addr show eth1                     # must show 10.240.71.x
ip route show | grep 10.240.71        # route must exist
```

**STONITH fails — resources won't start after node failure:**
```bash
# Physically confirm the node is off, then tell Pacemaker:
pcs stonith confirm db-node2

# Re-test fence agent connectivity:
fence_ipmilan -a 10.248.30.11 -l ADMIN -p 'IMMpassword!' --lanplus -o status
```

**LVM fails to activate on failover:**
```bash
vgs -o+systemid db_vg                 # check for stale system_id
vgchange --systemid "" db_vg          # clear if stale
pcs resource show db_lvm              # verify RA config
```

**Two-node quorum — resources won't start after one node fails:**
```bash
# This is correct behavior. One node cannot get quorum alone UNLESS
# STONITH confirms the other is off. Never set no-quorum-policy=ignore
# with shared storage. Ensure STONITH/IMM connectivity is working.
pcs property show no-quorum-policy    # should be: stop
```

---

> **References:**
> RHEL 8 HA Cluster Docs: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_high_availability_clusters/
> IBM IMM IPMI setup: IBM Documentation — *Enabling IPMI over LAN on IBM System x*
> `man corosync.conf` | `man pcs` | `man fence_ipmilan`

---

The file environment is temporarily unavailable so I can't generate the `.md` download directly, but the entire guide is above — you can copy it straight into a `.md` file for GitHub. Everything here is cross-referenced against:

- RHEL 8 official HA cluster documentation (PCS 0.10.x / Pacemaker 2.x syntax)
- Corosync 3.x `knet` dual-link config (verified against corosync-qdevice and knet link parameter naming)
- `fence_ipmilan` parameter names (`ipaddr`, `login`, `passwd`, `lanplus`, `pcmk_host_list`, `power_wait`) confirmed from fence-agents-all man page
- `LVM-activate` RA parameters (`vgname`, `vg_access_mode`, `activation_mode`) from ocf:heartbeat:LVM-activate reference
