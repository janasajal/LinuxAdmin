# 🛰️ Red Hat Satellite 6 — Complete Administration Guide

> **Author:** Sajal Jana
> **Based on:** RH403 v6.15 | Hands-On Lab Guide
> **Date:** March 2026

---

## Table of Contents

1. [Architecture & Key Concepts](#1-architecture--key-concepts)
2. [Installation Validation](#2-installation-validation)
3. [Manifest & Subscriptions](#3-manifest--subscriptions)
4. [Content Views & Lifecycle Environments](#4-content-views--lifecycle-environments)
5. [Sync Plans](#5-sync-plans)
6. [Activation Keys & Host Registration](#6-activation-keys--host-registration)
7. [Remote Execution (REX)](#7-remote-execution-rex)
8. [Errata Management](#8-errata-management)
9. [Users, Roles & RBAC](#9-users-roles--rbac)
10. [Capsule Server](#10-capsule-server)
11. [Backup & Recovery](#11-backup--recovery)
12. [Daily Admin Checklist & Quick Reference](#12-daily-admin-checklist--quick-reference)
13. [Lab Environment Reference](#13-lab-environment-reference)

---

## 1. Architecture & Key Concepts

### What is Red Hat Satellite?

Red Hat Satellite is an enterprise infrastructure management platform for RHEL environments. It provides centralized content management, host registration, patch management, provisioning, and compliance — all from a single interface.

```
Red Hat CDN
     |
     | (sync)
     ▼
Satellite Server  ←── Central Brain
     |
     | (sync)
     ▼
Capsule Server    ←── Remote Site Proxy
     |
     | (register/content)
     ▼
Content Hosts     ←── Managed RHEL Servers
```

### Core Components

| Component | Role |
|---|---|
| **Satellite Server** | Central brain — hosts Foreman, Pulp, Candlepin, PostgreSQL |
| **Capsule Server** | Remote proxy — mirrors content to branch offices/DMZs |
| **Content Host** | Managed RHEL server registered to Satellite |
| **Foreman** | Provisioning engine and CMDB |
| **Pulp** | Content storage and sync engine |
| **Candlepin** | Subscription management |
| **Dynflow** | Background task/job engine |
| **foreman-proxy** | Smart proxy for DNS, DHCP, TFTP, REX |

### Key Concepts

| Concept | Definition |
|---|---|
| **Organization** | Logical separation of content and hosts (e.g., per team/BU) |
| **Location** | Physical/logical site (e.g., DC-East, DC-West) |
| **Manifest** | Signed file from Red Hat portal containing subscription entitlements |
| **Product** | Collection of repositories (e.g., Red Hat Enterprise Linux for x86_64) |
| **Repository** | Actual RPM/content store synced from CDN |
| **Content View** | Frozen snapshot of repositories at a point in time |
| **Lifecycle Environment** | Promotion pipeline: Library → Dev → QA → Production |
| **Activation Key** | Auto-configuration token for host registration |
| **Host Collection** | Named group of hosts for bulk operations |
| **Simple Content Access (SCA)** | Modern mode — no per-host subscription attachment needed |

### Prerequisite Checks

| Check | Satellite | Capsule | Command |
|---|---|---|---|
| Min RAM | 20 GB | 12 GB | `free -g` |
| OS Version | RHEL 8 x86_64 | RHEL 8 x86_64 | `hostnamectl status` |
| Forward DNS | Required | Required | `ping -c1 $(hostname -f)` |
| Reverse DNS | Required | Required | `host <IP>` |
| NTP (chrony) | Required | Required | `systemctl status chronyd` |
| CDN Access | Required | Via Satellite | `ping -c1 cdn.redhat.com` |

---

## 2. Installation Validation

### Verify Satellite Services

```bash
satellite-maintain service list
satellite-maintain health check
```

All services should show `enabled` or `indirect`. All health checks should return `[OK]`.

### Key Services and Their Roles

| Service | Role |
|---|---|
| `foreman.service` | Core web app — UI and API |
| `httpd.service` | Apache web server |
| `postgresql.service` | Database |
| `pulpcore-api.service` | Content management API |
| `pulpcore-content.service` | Serves content to hosts |
| `pulpcore-worker.service` | Background sync/publish jobs |
| `tomcat.service` | Runs Candlepin (subscriptions) |
| `dynflow-sidekiq.service` | Background task engine |
| `redis.service` | Cache layer |

### Required Firewall Ports

```bash
firewall-cmd --list-all
```

| Port | Protocol | Purpose |
|---|---|---|
| `80/tcp` | HTTP | Content download redirect |
| `443/tcp` | HTTPS | Web UI, API, host registration |
| `5647/tcp` | AMQP | Capsule messaging (Qpid) |
| `8000/tcp` | HTTP | Kickstart file download (provisioning) |
| `8140/tcp` | TCP | Puppet agent communication |
| `9090/tcp` | HTTPS | Foreman Smart Proxy / Capsule API |
| `53/tcp+udp` | DNS | If Satellite manages DNS |
| `67/udp` | DHCP | If Satellite manages DHCP |
| `69/udp` | TFTP | PXE boot for provisioning |

> 💡 **NOTE:** NTP sync is critical. SSL certificates fail if clock drift exceeds 5 minutes between Satellite and managed hosts.

---

## 3. Manifest & Subscriptions

### What is a Manifest?

A manifest is a digitally signed `.zip` file downloaded from `access.redhat.com`. It contains subscription entitlements that allow Satellite to sync content from the Red Hat CDN on behalf of all managed hosts.

```
access.redhat.com
      |
      | (download manifest.zip)
      ▼
Your Satellite
      |
      | (syncs content on behalf of all hosts)
      ▼
Your 500 RHEL Servers (never touch the internet directly)
```

### Upload Manifest (Web UI)

1. Navigate to **Content → Subscriptions**
2. Click **Manage Manifest → Choose File** → select `manifest.zip`
3. Click **Upload** and wait for completion

### Manifest Commands

```bash
# Verify subscriptions imported
hammer subscription list \
  --organization "Default Organization"

# Refresh manifest (after adding subscriptions on portal)
hammer subscription refresh-manifest \
  --organization "Default Organization"

# Check manifest history
hammer subscription manifest-history \
  --organization "Default Organization"

# Check subscription expiry
hammer subscription list \
  --organization "Default Organization" \
  --fields "Name,End Date,Quantity"
```

### Enable Repositories

```bash
# Enable RHEL 8 BaseOS
hammer repository-set enable \
  --organization "Default Organization" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)" \
  --releasever "8" --basearch "x86_64"

# Enable RHEL 8 AppStream
hammer repository-set enable \
  --organization "Default Organization" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)" \
  --releasever "8" --basearch "x86_64"
```

> ⚠️ **WARN:** One manifest per Satellite. Never share a manifest between two Satellite servers.

> 💡 **TIP:** With SCA enabled, hosts automatically get content access. No per-host subscription attachment needed.

---

## 4. Content Views & Lifecycle Environments

### Why Content Views?

Content Views are **frozen snapshots** of your repositories. They prevent new (potentially buggy) content from automatically reaching production servers.

```
Synced Repos (always latest)
        |
        | (you decide when to publish)
        ▼
Content View v1.0  ←── January snapshot
Content View v2.0  ←── February snapshot
        |
        | (you decide what reaches prod)
        ▼
Your Servers see ONLY what YOU approve
```

### Create Lifecycle Environments

```bash
hammer lifecycle-environment create \
  --name "Dev" --prior "Library" --organization "Operations"

hammer lifecycle-environment create \
  --name "QA" --prior "Dev" --organization "Operations"

hammer lifecycle-environment create \
  --name "Production" --prior "QA" --organization "Operations"
```

### Content View Workflow

| Step | Action | Command |
|---|---|---|
| 1 | Create Content View | `hammer content-view create --name "RHEL8-CV" --organization "Ops"` |
| 2 | Add Repository | `hammer content-view add-repository --name "RHEL8-CV" --repository "BaseOS"` |
| 3 | Publish Version | `hammer content-view publish --name "RHEL8-CV" --organization "Ops"` |
| 4 | Promote to Dev | `hammer content-view version promote --to-lifecycle-environment "Dev"` |
| 5 | Promote to QA | `hammer content-view version promote --to-lifecycle-environment "QA"` |
| 6 | Promote to Prod | `hammer content-view version promote --to-lifecycle-environment "Production"` |

### Full Promote Command

```bash
hammer content-view version promote \
  --content-view "RHEL8-BaseOS-CV" \
  --version 1.0 \
  --to-lifecycle-environment "Production" \
  --organization "Default Organization"
```

### List & Verify

```bash
# List all content views
hammer content-view list --organization "Default Organization"

# Check which environments a CV is promoted to
hammer content-view info \
  --name "RHEL8-BaseOS-CV" \
  --organization "Default Organization" \
  --fields "Name,Lifecycle Environments"
```

> ⚠️ **WARN:** Publishing creates a snapshot in Library only. You MUST promote to make it available to hosts.

> 💡 **TIP:** Always promote Dev → QA → Production. Never skip stages, even for emergency patches.

---

## 5. Sync Plans

### What is a Sync Plan?

A Sync Plan is a scheduled job that automatically pulls latest content from the Red Hat CDN into Satellite's local storage — so you don't have to do it manually.

### Create and Assign a Sync Plan

```bash
# Create daily sync plan
hammer sync-plan create \
  --name "Daily Sync" \
  --interval daily \
  --sync-date "2026-03-02 02:00:00" \
  --enabled true \
  --organization "Default Organization"

# Assign to product
hammer product set-sync-plan \
  --name "Red Hat Enterprise Linux for x86_64" \
  --sync-plan "Daily Sync" \
  --organization "Default Organization"

# Verify sync plan is enabled
hammer sync-plan info \
  --name "Daily Sync" \
  --organization "Default Organization"

# Trigger manual sync
hammer product synchronize \
  --name "Red Hat Enterprise Linux for x86_64" \
  --organization "Default Organization"
```

### Recommended Sync Schedule

| Time | Task |
|---|---|
| 2:00 AM | Satellite syncs all products from Red Hat CDN |
| 4:00 AM | Capsules sync from Satellite |
| Sunday 1AM | Full backup of Satellite |
| Mon-Sat 1AM | Incremental backup |

### Sync Plan Intervals

| Interval | Use Case |
|---|---|
| `hourly` | Testing only |
| `daily` | Recommended for production |
| `weekly` | Low priority / large repos |

> ⚠️ **WARN:** Always verify sync plan is `enabled: true`. A disabled plan silently does nothing.

---

## 6. Activation Keys & Host Registration

### What is an Activation Key?

An Activation Key is a pre-configuration token. When a host registers using an activation key, it automatically gets assigned the correct Content View, Lifecycle Environment, Host Collections, and Repository access — no manual steps needed.

### Create Activation Key

```bash
# Create key
hammer activation-key create \
  --name "OperationsServers" \
  --organization "Operations" \
  --lifecycle-environment "Development" \
  --content-view "OperationsServerBase" \
  --unlimited-hosts

# Set release version
hammer activation-key update \
  --name "OperationsServers" \
  --organization "Operations" \
  --release-version "9"

# Add host collection
hammer activation-key add-host-collection \
  --name "OperationsServers" \
  --host-collection "OpsServers" \
  --organization "Operations"

# Enable a repository override
hammer activation-key content-override \
  --name "OperationsServers" \
  --organization "Operations" \
  --content-label "satellite-client-6-for-rhel-9-x86_64-rpms" \
  --value 1

# List activation keys
hammer activation-key list --organization "Operations"
```

### Register a Host

```bash
# Step 1 — Install Satellite CA certificate on host
dnf localinstall \
  http://satellite.lab.example.com/pub/katello-ca-consumer-latest.noarch.rpm

# Step 2 — Register (use org LABEL not display name)
subscription-manager register \
  --org Operations \
  --activationkey OperationsServers

# Step 3 — Verify SCA mode (org_environment = SCA active)
cat /var/lib/rhsm/cache/content_access_mode.json | python -m json.tool

# Step 4 — Install Katello agent
dnf install katello-agent -y
```

### Clean Up Old Registration

```bash
dnf clean all
rm -rf /var/cache/dnf/
subscription-manager remove --all
subscription-manager unregister
subscription-manager clean
```

### Registration Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| HTTP 401 Unauthorized | Pointing to wrong Satellite | Install `katello-ca-consumer` RPM |
| Organization does not exist | Wrong org name | Use label: `hammer organization list` |
| Activation key not found | Key in wrong org | Check: `hammer activation-key list --organization "Org"` |
| HTTP 500 (no env/CV) | Activation key misconfigured | `hammer activation-key update --lifecycle-environment` |
| Permission denied (publickey) | REX SSH key missing | Manually add `foreman-proxy` pub key to host |

> 💡 **TIP:** Always use the org **LABEL** (no spaces), not the display name when registering hosts.

> ⚠️ **WARN:** Every activation key MUST have an Organization, Lifecycle Environment, and Content View assigned. Missing any one of these causes registration to fail.

---

## 7. Remote Execution (REX)

### How REX Works

REX uses SSH key-based authentication to run jobs on managed hosts without logging into each one individually. Satellite's `foreman-proxy` SSH public key must exist in `/root/.ssh/authorized_keys` on each target host.

### REX Commands

```bash
# Run command on single host
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="uptime" \
  --search-query "name = serverc.lab.example.com"

# Run on entire host collection
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="df -h" \
  --search-query "host_collection = OpsServers"

# Run on all hosts in an org
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="uname -r" \
  --search-query "organization = Operations"

# Check job status
hammer job-invocation list

# Get job output
hammer job-invocation output \
  --id <JOB_ID> \
  --host serverc.lab.example.com
```

### Fix REX SSH Key Issue

```bash
# On Satellite — get the public key
cat /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub

# On target host — add the key manually
mkdir -p /root/.ssh && chmod 700 /root/.ssh
echo "<PASTE_KEY_HERE>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

# Test SSH connectivity from Satellite
ssh -i /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy \
  root@serverc.lab.example.com 'hostname'
```

### Search Query Reference

| Target | Search Query |
|---|---|
| Single host | `name = serverc.lab.example.com` |
| Host collection | `host_collection = OpsServers` |
| All RHEL 9 hosts | `os = RHEL 9` |
| By lifecycle env | `lifecycle_environment = Production` |
| By organization | `organization = Operations` |
| Combined | `host_collection = OpsServers and os = RHEL 9` |

---

## 8. Errata Management

### Errata Types & Priority

| Type | Priority | Action |
|---|---|---|
| Security (Critical/Important) | 🔴 Immediate | Patch same day |
| Security (Moderate) | 🟠 High | Patch within 7 days |
| Bugfix | 🟡 Medium | Next maintenance window |
| Enhancement | 🟢 Low | Next quarterly update |

### Apply Errata Commands

```bash
# List errata on a host
hammer host errata list --host serverc.lab.example.com

# Apply security errata to single host
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "name = serverc.lab.example.com" \
  --inputs errata=security

# Apply ALL errata to entire org
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "organization = Operations"

# Apply specific errata by ID
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "name = serverc.lab.example.com" \
  --inputs errata=RHSA-2026:1234

# Check if reboot needed after patching
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="needs-restarting -r; echo Exit:$?" \
  --search-query "name = serverc.lab.example.com"
```

### Patch Tuesday Workflow

```
Monday AM  : Sync latest content from CDN
Tuesday AM : Publish new Content View version
Wed-Fri    : Promote to Dev → Dev team tests
Next Monday: Promote to QA → QA validates
Next Friday: Promote to Production ✅
```

> 💡 **TIP:** Always verify errata count = 0 after patching with `hammer host errata list`.

---

## 9. Users, Roles & RBAC

### Why RBAC?

Role-Based Access Control ensures users only have access to what they need. This prevents accidental deletions, unauthorized changes, and improves security audit compliance.

### Create User and Role

```bash
# Create user
hammer user create \
  --login jradmin \
  --password redhat123 \
  --firstname Junior \
  --lastname Admin \
  --mail jradmin@example.com \
  --organizations "Operations" \
  --auth-source-id 1

# Create role
hammer role create --name "Junior Admin"

# Add permissions to role
hammer filter create \
  --role "Junior Admin" \
  --permissions "view_hosts,edit_hosts"

# Add content view permissions
hammer filter create \
  --role "Junior Admin" \
  --permissions "view_content_views"

# Assign role to user
hammer user add-role \
  --login jradmin \
  --role "Junior Admin"

# Verify user roles
hammer user info --login jradmin --fields "Login,Roles"

# List all roles
hammer role list

# List permissions for a resource
hammer permission list --resource-type Host
```

### Built-in Roles Reference

| Role | Best For |
|---|---|
| **Viewer** | Read-only access — helpdesk |
| **Site Manager** | Manage one location |
| **Organization Admin** | Full org access — team lead |
| **Manager** | Most admin tasks |
| **System Admin** | Full Satellite access — senior admin |

> 💡 **TIP:** Follow least privilege principle — give users ONLY the permissions they need.

> 💡 **TIP:** In production, integrate with LDAP/Active Directory via **Administer → LDAP Authentication** for centralized user management.

---

## 10. Capsule Server

### What Does a Capsule Do?

A Capsule Server is a smart proxy deployed at remote/branch sites. It mirrors content locally so remote hosts don't need to communicate with Satellite HQ over WAN for every package download.

```
Satellite HQ (headquarters)
      |
      | WAN (only metadata/sync traffic)
      ▼
Capsule (branch office)
      |
      | LAN (fast local traffic)
      ▼
Branch Office Hosts
```

### Capsule Services

| Service | Function |
|---|---|
| Pulp Content | Local RPM repository mirror |
| Registration | Proxy for host registration |
| Remote Execution | REX proxy for remote jobs |
| DNS/DHCP/TFTP | Infrastructure services for provisioning |
| Templates | Kickstart/provisioning templates |
| OpenSCAP | Compliance scan results proxy |

### Capsule Commands

```bash
# List all capsules
hammer capsule list

# Get capsule details
hammer capsule info --name capsule.lab.example.com

# Add lifecycle environment to capsule
hammer capsule content add-lifecycle-environment \
  --name capsule.lab.example.com \
  --organization "Operations" \
  --lifecycle-environment "Production"

# Sync content to capsule
hammer capsule content synchronize \
  --name capsule.lab.example.com

# Check sync status
hammer capsule content sync-status \
  --name capsule.lab.example.com

# List capsule lifecycle environments
hammer capsule content lifecycle-environments \
  --name capsule.lab.example.com
```

### Capsule Sizing Guide

| Hosts Managed | RAM | Disk | CPU |
|---|---|---|---|
| Up to 500 | 12 GB | 500 GB | 4 cores |
| 500 – 2000 | 16 GB | 1 TB | 8 cores |
| 2000 – 5000 | 32 GB | 2 TB | 16 cores |

> ⚠️ **WARN:** Always sync Satellite FIRST, then sync Capsules. Never the other way around.

> ⚠️ **WARN:** Capsule and Satellite must run the same RHEL major version. Mismatched versions cause mysterious failures.

---

## 11. Backup & Recovery

### What Gets Backed Up

| Component | Location | Est. Size |
|---|---|---|
| PostgreSQL DB | `/var/lib/pgsql/` | 1–10 GB |
| Candlepin DB | Included in DB backup | 100–500 MB |
| Config files | `/etc/foreman/`, `/etc/pulp/` | 10–50 MB |
| SSL certificates | `/etc/pki/` | 1–5 MB |
| Pulp content (RPMs) | `/var/lib/pulp/` | 50 GB – 2 TB |

### Backup Commands

```bash
# Full online backup (Satellite stays running)
satellite-maintain backup online \
  --assumeyes \
  /backup/satellite

# Skip pulp content (faster — lab use only)
satellite-maintain backup online \
  --assumeyes \
  --skip-pulp-content \
  /backup/satellite

# Incremental backup (changes since last backup)
satellite-maintain backup online \
  --assumeyes \
  --incremental /backup/satellite/PREVIOUS_BACKUP_DIR \
  /backup/satellite

# Offline backup (most consistent — use during maintenance)
satellite-maintain backup offline \
  --assumeyes \
  /backup/satellite

# Restore from backup (WARNING: overwrites current config!)
satellite-maintain restore --assumeyes /backup/satellite/BACKUP_DIR

# Verify backup metadata
cat /backup/satellite/*/metadata.yml

# Check backup size
du -sh /backup/satellite/*/
```

### Recommended Backup Schedule

| Schedule | Type | Notes |
|---|---|---|
| Sunday 1AM | Full backup | All components including pulp |
| Mon–Sat 1AM | Incremental | Changes only, much faster |
| Monthly | Offsite copy | `rsync` to remote backup server |
| Quarterly | Test restore | Restore to test server and verify |

> ⚠️ **WARN:** A backup you have never tested is NOT a backup. Test restores quarterly!

> ⚠️ **WARN:** In production, ALWAYS backup pulp content. `--skip-pulp-content` is for lab use only.

---

## 12. Daily Admin Checklist & Quick Reference

### Morning Health Check

```bash
# 1. Verify all services running
satellite-maintain service list

# 2. Run full health check
satellite-maintain health check

# 3. Check overnight sync status
# Web UI: Content → Sync Status

# 4. Check for failed jobs
hammer job-invocation list

# 5. Check disk space
df -h /var/lib/pulp /var/lib/pgsql

# 6. Check paused tasks
# Web UI: Monitor → Jobs → filter by paused
```

### Essential Hammer Commands

```bash
# ── Organizations & Hosts ─────────────────────────────────────
hammer organization list
hammer host list --organization "Org"
hammer host info --name hostname

# ── Content Management ────────────────────────────────────────
hammer content-view list --organization "Org"
hammer content-view publish --name "CV" --organization "Org"
hammer content-view version promote \
  --content-view "CV" --version 1.0 \
  --to-lifecycle-environment "Env" --organization "Org"
hammer lifecycle-environment list --organization "Org"
hammer product synchronize --name "Product" --organization "Org"

# ── Activation Keys & Registration ───────────────────────────
hammer activation-key list --organization "Org"
hammer activation-key create --name "Key" --organization "Org" \
  --lifecycle-environment "Env" --content-view "CV" --unlimited-hosts

# ── Errata & Patching ─────────────────────────────────────────
hammer host errata list --host hostname
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "host_collection = Collection"

# ── Remote Execution ──────────────────────────────────────────
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="CMD" \
  --search-query "name = hostname"
hammer job-invocation list
hammer job-invocation output --id ID --host hostname

# ── Capsule ───────────────────────────────────────────────────
hammer capsule list
hammer capsule content synchronize --name capsule.lab.example.com

# ── Users & Roles ─────────────────────────────────────────────
hammer user list
hammer role list
hammer user add-role --login username --role "Role Name"

# ── Backup ────────────────────────────────────────────────────
satellite-maintain backup online --assumeyes /backup/satellite
satellite-maintain health check
```

### Common Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| HTTP 401 on registration | No Satellite CA cert | Install `katello-ca-consumer` RPM |
| Activation key not found | Wrong organization | `hammer activation-key list` for each org |
| REX permission denied | SSH key not distributed | Add `foreman-proxy` pub key to host manually |
| Sync failed | Disk full or CDN unreachable | `df -h /var/lib/pulp`; check firewall |
| Web UI slow / timeouts | Stuck tasks or low RAM | `satellite-maintain health check` |
| Host sees wrong content | CV not promoted to env | `hammer content-view version promote` |
| Manifest refresh fails | SCA / disconnected env | Expected in disconnected lab environments |
| Services not starting | Disk full or DB issue | `df -h`; check `journalctl -u postgresql` |

---

## 13. Lab Environment Reference

### Lab Machines (RH403 v6.15)

| Hostname | IP Address | Role |
|---|---|---|
| `workstation.lab.example.com` | 172.25.250.9 | Student graphical workstation |
| `satellite.lab.example.com` | 172.25.250.16 | Satellite Server |
| `capsule.lab.example.com` | 172.25.250.17 | Capsule Server |
| `servera.lab.example.com` | 172.25.250.10 | Content host A |
| `serverb.lab.example.com` | 172.25.250.11 | Content host B |
| `serverc.lab.example.com` | 172.25.250.12 | Content host C |
| `serverd.lab.example.com` | 172.25.250.13 | Content host D |
| `servere.lab.example.com` | 172.25.250.14 | Content host E |
| `utility.lab.example.com` | 172.25.250.220 | DNS, LDAP and utility services |
| `classroom.example.com` | 172.25.254.254 | Classroom materials server |

### Lab Credentials

| System | Username | Password |
|---|---|---|
| All hosts (SSH) | `student` | `student` |
| Root on all hosts | `root` | `redhat` |
| Satellite Web UI | `admin` | `redhat` |
| Satellite CLI (hammer) | `admin` | `redhat` |

### Terminal Prompt Reference

```bash
[student@workstation ~]$   # Your main desktop — start all labs here
[root@satellite ~]#        # Satellite — run hammer commands here
[root@capsule ~]#          # Capsule server
[root@servera ~]#          # Content host A
[root@serverc ~]#          # Content host C
[root@serverd ~]#          # Content host D
```

> ⚠️ **WARN:** Always check your terminal prompt before running commands. `subscription-manager` runs on **content hosts**, `hammer` runs on the **Satellite server**.

---

## Real-World Lessons Learned

These are the most common mistakes made in production — and how to fix them:

1. **Always verify the Organization** in the top-left corner of the Web UI before creating anything. Working in the wrong org is the #1 mistake.

2. **Use org LABEL not name** — Labels have no spaces (`Default_Organization`, `Operations`). Names can have spaces and cause 401 errors.

3. **Content View must be promoted** to a lifecycle environment before hosts in that environment can see new content. Publishing to Library is not enough.

4. **Every Activation Key needs** Organization + Lifecycle Environment + Content View. Missing any one causes registration to fail with HTTP 500.

5. **REX needs SSH keys** on target hosts. If you see `Permission denied (publickey)`, manually copy the `foreman-proxy` public key.

6. **DNS must work in both directions** — forward (FQDN → IP) and reverse (IP → FQDN). Broken reverse DNS causes SSL certificate errors and host registration failures.

7. **Monitor disk space on `/var/lib/pulp`** — This is where all RPMs are stored. It can fill up fast in production (500 GB – 2 TB+). A full disk silently breaks syncs.

8. **Test your backups** — Run a restore on a test system quarterly. Discovering a corrupt backup during an actual disaster is the worst time to find out.

---

*Author: Sajal Jana | Red Hat Satellite 6 Administration (RH403) v6.15 | March 2026*
