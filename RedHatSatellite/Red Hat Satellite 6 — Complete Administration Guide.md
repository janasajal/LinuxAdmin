# Red Hat Satellite 6 — Administration Guide

**Author:** Sajal Jana  
**Based on:** RH403 v6.15 | Hands-On Lab Guide  
**Updated:** March 2026

---

## Table of Contents

1. [What is Red Hat Satellite?](#1-what-is-red-hat-satellite)
2. [Verifying Your Installation](#2-verifying-your-installation)
3. [Manifests & Subscriptions](#3-manifests--subscriptions)
4. [Content Views & Lifecycle Environments](#4-content-views--lifecycle-environments)
5. [Sync Plans](#5-sync-plans)
6. [Activation Keys & Host Registration](#6-activation-keys--host-registration)
7. [Remote Execution (REX)](#7-remote-execution-rex)
8. [Errata & Patch Management](#8-errata--patch-management)
9. [Users, Roles & Access Control](#9-users-roles--access-control)
10. [Capsule Servers](#10-capsule-servers)
11. [Backup & Recovery](#11-backup--recovery)
12. [Daily Admin Checklist & Reference Commands](#12-daily-admin-checklist--reference-commands)
13. [Lab Environment Reference](#13-lab-environment-reference)

---

## 1. What is Red Hat Satellite?

Red Hat Satellite is an enterprise platform for managing RHEL servers at scale. Instead of logging into each server individually, Satellite gives you a single control plane to handle:

- **Content delivery** — decide which packages and updates reach which servers
- **Host registration** — bring new servers under management automatically
- **Patch management** — apply security updates across hundreds of hosts at once
- **Provisioning** — build new servers from scratch via PXE boot
- **Compliance** — audit and enforce security baselines

### How the Components Fit Together

```
Red Hat CDN  (source of truth for packages)
     │
     ▼
Satellite Server  ←── Central brain
     │
     ▼
Capsule Server    ←── Remote site proxy (optional)
     │
     ▼
Content Hosts     ←── Your managed RHEL servers
```

### Core Components at a Glance

| Component | What It Does |
|---|---|
| **Satellite Server** | The central hub — hosts the web UI, API, and all core services |
| **Capsule Server** | A remote proxy that mirrors content to branch offices or DMZs |
| **Content Host** | Any RHEL server registered and managed by Satellite |
| **Foreman** | Handles provisioning and acts as a configuration database (CMDB) |
| **Pulp** | Stores and syncs RPM content from the CDN |
| **Candlepin** | Manages subscription entitlements |
| **Dynflow** | Runs background jobs and tasks |
| **foreman-proxy** | Smart proxy for DNS, DHCP, TFTP, and remote execution |

### Key Concepts You Must Understand

| Concept | What It Means in Plain English |
|---|---|
| **Organization** | A logical tenant — separates content, hosts, and settings by team or business unit |
| **Location** | Represents a physical or logical site (e.g., DC-East, DC-West) |
| **Manifest** | A signed file from Red Hat that proves you own subscriptions, allowing content sync |
| **Product** | A bundle of repositories (e.g., RHEL for x86_64 contains BaseOS + AppStream) |
| **Repository** | The actual store of RPM packages synced from the CDN |
| **Content View** | A frozen snapshot of repositories — your servers only see what you approve |
| **Lifecycle Environment** | The promotion pipeline: Library → Dev → QA → Production |
| **Activation Key** | A token that auto-configures a host when it registers |
| **Host Collection** | A named group of hosts for running bulk operations |
| **Simple Content Access (SCA)** | Modern subscription mode — no need to manually attach subscriptions per host |

### Pre-Installation Requirements

Before installing, verify these on both Satellite and Capsule:

| Requirement | Satellite | Capsule | How to Check |
|---|---|---|---|
| RAM | 20 GB min | 12 GB min | `free -g` |
| OS | RHEL 8 x86_64 | RHEL 8 x86_64 | `hostnamectl status` |
| Forward DNS | ✅ Required | ✅ Required | `ping -c1 $(hostname -f)` |
| Reverse DNS | ✅ Required | ✅ Required | `host <IP>` |
| NTP (chrony) | ✅ Required | ✅ Required | `systemctl status chronyd` |
| CDN access | ✅ Required | Via Satellite only | `ping -c1 cdn.redhat.com` |

---

## 2. Verifying Your Installation

After installing Satellite, confirm everything is healthy before doing anything else.

```bash
# Check all service statuses
satellite-maintain service list

# Run the built-in health checks
satellite-maintain health check
```

All services should show `enabled` or `indirect`. All health checks should return `[OK]`.

### What Each Service Does

| Service | Purpose |
|---|---|
| `foreman.service` | Core web application — UI and API |
| `httpd.service` | Apache web server (fronts the UI) |
| `postgresql.service` | Database for all Satellite data |
| `pulpcore-api.service` | API for content management |
| `pulpcore-content.service` | Serves packages to hosts |
| `pulpcore-worker.service` | Runs background sync and publish jobs |
| `tomcat.service` | Runs Candlepin (subscription engine) |
| `dynflow-sidekiq.service` | Background task processing |
| `redis.service` | Caching layer |

### Required Firewall Ports

```bash
firewall-cmd --list-all
```

| Port | Protocol | Used For |
|---|---|---|
| 80/tcp | HTTP | Content download redirects |
| 443/tcp | HTTPS | Web UI, API, host registration |
| 5647/tcp | AMQP | Capsule messaging |
| 8000/tcp | HTTP | Kickstart file delivery (provisioning) |
| 8140/tcp | TCP | Puppet agent communication |
| 9090/tcp | HTTPS | Capsule/Smart Proxy API |
| 53/tcp+udp | DNS | Only if Satellite manages DNS |
| 67/udp | DHCP | Only if Satellite manages DHCP |
| 69/udp | TFTP | PXE boot for provisioning |

> **⚠️ Important:** Keep clocks in sync. SSL certificates break if the clock difference between Satellite and a managed host exceeds 5 minutes.

---

## 3. Manifests & Subscriptions

### What is a Manifest?

A manifest is a digitally signed `.zip` file you download from `access.redhat.com`. It tells Satellite which Red Hat subscriptions you own, allowing it to sync content from the CDN on behalf of all your servers — your servers never need to touch the internet directly.

```
access.redhat.com
      │
      │  (download manifest.zip once)
      ▼
Satellite Server
      │
      │  (syncs content for everyone)
      ▼
All 500 of your RHEL servers
```

### Upload the Manifest (Web UI)

1. Go to **Content → Subscriptions**
2. Click **Manage Manifest → Choose File**, select `manifest.zip`
3. Click **Upload** and wait for it to finish

### Common Manifest Commands

```bash
# See what subscriptions were imported
hammer subscription list \
  --organization "Default Organization"

# Pull in changes after adding subscriptions on the portal
hammer subscription refresh-manifest \
  --organization "Default Organization"

# View manifest upload history
hammer subscription manifest-history \
  --organization "Default Organization"

# Check subscription expiry dates
hammer subscription list \
  --organization "Default Organization" \
  --fields "Name,End Date,Quantity"
```

### Enable Repositories

Enabling a repository tells Satellite to make it available for syncing. You must do this before content will sync.

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

> **⚠️ Warning:** One manifest per Satellite. Never share a manifest between two Satellite servers — it breaks both.

> **💡 Tip:** With Simple Content Access (SCA) enabled, hosts get content automatically upon registration. No manual subscription attachment needed.

---

## 4. Content Views & Lifecycle Environments

### Why You Need Content Views

Without Content Views, every package update Red Hat releases would immediately be available to your production servers — including potentially buggy ones. Content Views give you control.

```
Red Hat CDN (constantly publishing new packages)
        │
        │  (you control when you pull a snapshot)
        ▼
Content View v1.0  ←── Tested, stable, January packages
Content View v2.0  ←── February packages (still in testing)
        │
        │  (you promote when you're ready)
        ▼
Production Servers see ONLY what you've approved
```

### Create the Promotion Pipeline

The lifecycle pipeline lets you test changes in Dev before they ever reach Production.

```bash
hammer lifecycle-environment create \
  --name "Dev" --prior "Library" --organization "Operations"

hammer lifecycle-environment create \
  --name "QA" --prior "Dev" --organization "Operations"

hammer lifecycle-environment create \
  --name "Production" --prior "QA" --organization "Operations"
```

### Content View Workflow — Step by Step

| Step | What You're Doing | Command |
|---|---|---|
| 1 | Create an empty Content View | `hammer content-view create --name "RHEL8-CV" --organization "Ops"` |
| 2 | Add a repository to it | `hammer content-view add-repository --name "RHEL8-CV" --repository "BaseOS"` |
| 3 | Publish a version (snapshot) | `hammer content-view publish --name "RHEL8-CV" --organization "Ops"` |
| 4 | Promote to Dev | `hammer content-view version promote --to-lifecycle-environment "Dev"` |
| 5 | Promote to QA | `hammer content-view version promote --to-lifecycle-environment "QA"` |
| 6 | Promote to Production | `hammer content-view version promote --to-lifecycle-environment "Production"` |

### Promote a Specific Version

```bash
hammer content-view version promote \
  --content-view "RHEL8-BaseOS-CV" \
  --version 1.0 \
  --to-lifecycle-environment "Production" \
  --organization "Default Organization"
```

### Verify Your Content Views

```bash
# List all content views
hammer content-view list --organization "Default Organization"

# Check where a specific CV has been promoted
hammer content-view info \
  --name "RHEL8-BaseOS-CV" \
  --organization "Default Organization" \
  --fields "Name,Lifecycle Environments"
```

> **⚠️ Warning:** Publishing creates a snapshot in **Library only**. You must promote it to a lifecycle environment before any hosts can see it.

> **💡 Tip:** Always follow the Dev → QA → Production path. Even for emergency patches — skipping stages causes problems that are hard to trace later.

---

## 5. Sync Plans

### What is a Sync Plan?

A Sync Plan is a scheduled job that automatically pulls the latest packages from the Red Hat CDN into Satellite's local storage. Without one, your content never updates unless you trigger syncs manually.

### Create and Assign a Sync Plan

```bash
# Create a plan that runs daily at 2 AM
hammer sync-plan create \
  --name "Daily Sync" \
  --interval daily \
  --sync-date "2026-03-02 02:00:00" \
  --enabled true \
  --organization "Default Organization"

# Attach it to a product
hammer product set-sync-plan \
  --name "Red Hat Enterprise Linux for x86_64" \
  --sync-plan "Daily Sync" \
  --organization "Default Organization"

# Confirm it's enabled
hammer sync-plan info \
  --name "Daily Sync" \
  --organization "Default Organization"

# Trigger a manual sync right now
hammer product synchronize \
  --name "Red Hat Enterprise Linux for x86_64" \
  --organization "Default Organization"
```

### Recommended Sync Schedule

| Time | Task |
|---|---|
| 2:00 AM | Satellite syncs all products from the CDN |
| 4:00 AM | Capsules sync from Satellite |
| Sunday 1:00 AM | Full Satellite backup |
| Mon–Sat 1:00 AM | Incremental backup |

### Sync Interval Options

| Interval | When to Use |
|---|---|
| `hourly` | Testing only — not for production |
| `daily` | Recommended default |
| `weekly` | Low-priority repos or large datasets |

> **⚠️ Warning:** Always confirm the sync plan is `enabled: true`. A disabled plan fails silently — you won't notice until your content is weeks out of date.

---

## 6. Activation Keys & Host Registration

### What is an Activation Key?

An Activation Key is a pre-configured token you hand to a new server during registration. When the server registers using that key, it automatically receives the correct Content View, Lifecycle Environment, Host Collection membership, and repository access — no manual configuration needed.

### Create an Activation Key

```bash
# Create the key
hammer activation-key create \
  --name "OperationsServers" \
  --organization "Operations" \
  --lifecycle-environment "Development" \
  --content-view "OperationsServerBase" \
  --unlimited-hosts

# Set the RHEL release version
hammer activation-key update \
  --name "OperationsServers" \
  --organization "Operations" \
  --release-version "9"

# Add a host collection
hammer activation-key add-host-collection \
  --name "OperationsServers" \
  --host-collection "OpsServers" \
  --organization "Operations"

# Enable a specific repository for this key
hammer activation-key content-override \
  --name "OperationsServers" \
  --organization "Operations" \
  --content-label "satellite-client-6-for-rhel-9-x86_64-rpms" \
  --value 1

# List all activation keys
hammer activation-key list --organization "Operations"
```

### Register a New Host

Run these commands **on the host you want to register**, not on Satellite:

```bash
# Step 1 — Trust Satellite's SSL certificate
dnf localinstall \
  http://satellite.lab.example.com/pub/katello-ca-consumer-latest.noarch.rpm

# Step 2 — Register using the activation key (use org LABEL, not display name)
subscription-manager register \
  --org Operations \
  --activationkey OperationsServers

# Step 3 — Verify SCA mode is active
cat /var/lib/rhsm/cache/content_access_mode.json | python -m json.tool

# Step 4 — Install the Katello agent
dnf install katello-agent -y
```

### Clean Up a Host Before Re-registering

```bash
dnf clean all
rm -rf /var/cache/dnf/
subscription-manager remove --all
subscription-manager unregister
subscription-manager clean
```

### Registration Error Reference

| Error | Root Cause | Solution |
|---|---|---|
| HTTP 401 Unauthorized | Missing Satellite CA cert | Install the `katello-ca-consumer` RPM |
| "Organization does not exist" | Wrong org name used | Use org label: `hammer organization list` |
| "Activation key not found" | Key is in a different org | Check: `hammer activation-key list --organization "Org"` |
| HTTP 500 | Activation key missing CV or environment | `hammer activation-key update --lifecycle-environment` |
| REX "Permission denied (publickey)" | SSH key not on host | Add the `foreman-proxy` public key manually |

> **💡 Tip:** Always use the org **LABEL** (no spaces, e.g., `Default_Organization`) — not the display name. Using the display name with spaces causes 401 errors.

> **⚠️ Warning:** Every activation key must have an Organization, Lifecycle Environment, and Content View. Missing any one causes registration to fail.

---

## 7. Remote Execution (REX)

### How REX Works

REX lets you run commands on managed hosts without SSH-ing into each one manually. Satellite's `foreman-proxy` service has an SSH key pair. Its public key must exist in `/root/.ssh/authorized_keys` on each target host. Once in place, you can run jobs from Satellite against any number of hosts simultaneously.

### Running Jobs

```bash
# Run a command on a single host
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="uptime" \
  --search-query "name = serverc.lab.example.com"

# Run on all hosts in a collection
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="df -h" \
  --search-query "host_collection = OpsServers"

# Run across an entire organization
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="uname -r" \
  --search-query "organization = Operations"

# Check job status
hammer job-invocation list

# View output from a completed job
hammer job-invocation output \
  --id <JOB_ID> \
  --host serverc.lab.example.com
```

### Fix the "Permission Denied" SSH Error

```bash
# On Satellite — copy the public key
cat /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub

# On the target host — add it manually
mkdir -p /root/.ssh && chmod 700 /root/.ssh
echo "<PASTE_KEY_HERE>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

# Test the connection from Satellite
ssh -i /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy \
  root@serverc.lab.example.com 'hostname'
```

### Search Query Reference

| Target Scope | Search Query Syntax |
|---|---|
| Single host | `name = serverc.lab.example.com` |
| Host collection | `host_collection = OpsServers` |
| All RHEL 9 hosts | `os = RHEL 9` |
| By lifecycle environment | `lifecycle_environment = Production` |
| By organization | `organization = Operations` |
| Combined filter | `host_collection = OpsServers and os = RHEL 9` |

---

## 8. Errata & Patch Management
In Red Hat Satellite, **errata** are official update notices from Red Hat that package one or more fixes into a named advisory. There are three types:

- **Security (RHSA)** — fixes vulnerabilities; highest priority
- **Bugfix (RHBA)** — fixes software defects
- **Enhancement (RHEA)** — adds new features or improvements

### How It Works

When Red Hat releases a fix, it publishes an errata advisory (e.g., `RHSA-2026:1234`). Satellite pulls this in during a content sync. You can then see exactly which of your hosts are missing which advisories — and apply them in bulk without touching each server individually.
### Basic Workflow

1. **Sync content** — Satellite pulls latest packages and errata from the CDN
2. **Publish a new Content View version** — captures the updated packages in a snapshot
3. **Promote through environments** — Dev → QA → Production at your own pace
4. **Apply errata to hosts** — via Remote Execution (REX), targeted by host, collection, or org

### Errata Priority Guide

| Errata Type | Priority | Target Response Time |
|---|---|---|
| Security — Critical / Important | 🔴 Urgent | Same day |
| Security — Moderate | 🟠 High | Within 7 days |
| Bugfix | 🟡 Medium | Next maintenance window |
| Enhancement | 🟢 Low | Next quarterly cycle |

### Applying Errata

```bash
# See what errata a host needs
hammer host errata list --host serverc.lab.example.com

# Apply all security errata to a single host
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "name = serverc.lab.example.com" \
  --inputs errata=security

# Apply all errata across an entire org
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "organization = Operations"

# Apply a specific errata by ID
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "name = serverc.lab.example.com" \
  --inputs errata=RHSA-2026:1234

# Check if a reboot is needed after patching
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="needs-restarting -r; echo Exit:$?" \
  --search-query "name = serverc.lab.example.com"
```

### Recommended Patch Cadence

```
Monday AM     Sync latest content from CDN
Tuesday AM    Publish new Content View version
Wed – Fri     Promote to Dev → Dev team tests
Next Monday   Promote to QA → QA team validates
Next Friday   Promote to Production ✅
```

> **💡 Tip:** After patching, always run `hammer host errata list` to confirm errata count is zero.

---

## 9. Users, Roles & Access Control

### Why RBAC Matters

Role-Based Access Control (RBAC) prevents accidents and unauthorized changes. A junior admin should be able to see host status but not delete content views. RBAC lets you enforce that precisely.

### Create a User and Assign a Role

```bash
# Create a new user
hammer user create \
  --login jradmin \
  --password redhat123 \
  --firstname Junior \
  --lastname Admin \
  --mail jradmin@example.com \
  --organizations "Operations" \
  --auth-source-id 1

# Create a custom role
hammer role create --name "Junior Admin"

# Grant permissions to the role
hammer filter create \
  --role "Junior Admin" \
  --permissions "view_hosts,edit_hosts"

hammer filter create \
  --role "Junior Admin" \
  --permissions "view_content_views"

# Assign the role to the user
hammer user add-role \
  --login jradmin \
  --role "Junior Admin"

# Verify the user's roles
hammer user info --login jradmin --fields "Login,Roles"

# List all defined roles
hammer role list

# See what permissions exist for a resource type
hammer permission list --resource-type Host
```

### Built-in Roles Reference

| Role | Best For |
|---|---|
| **Viewer** | Read-only — good for helpdesk staff |
| **Site Manager** | Full control of a single location |
| **Organization Admin** | Full control of one organization — team leads |
| **Manager** | Most admin tasks across the system |
| **System Admin** | Full Satellite access — senior admins only |

> **💡 Tip:** Always follow the principle of least privilege — give users only the access they actually need.

> **💡 Tip:** In production, connect Satellite to LDAP/Active Directory via **Administer → LDAP Authentication** to avoid managing users in two places.

---

## 10. Capsule Servers

### What is a Capsule For?

A Capsule Server is a local mirror for remote sites. Without a Capsule, every server in a branch office would pull packages from Satellite HQ over the WAN — slow and expensive. A Capsule sits in the branch office, holds a local copy of content, and handles registration and patching locally.

```
Satellite HQ
      │
      │  WAN — only sync/metadata traffic
      ▼
Capsule (branch office)
      │
      │  LAN — fast, local
      ▼
Branch Office Servers
```

### What a Capsule Can Handle

| Capability | What It Does |
|---|---|
| Pulp Content | Local RPM repository mirror |
| Host Registration | Proxies registration requests to Satellite |
| Remote Execution | Runs REX jobs for local hosts |
| DNS / DHCP / TFTP | Infrastructure services for local provisioning |
| Kickstart Templates | Serves provisioning templates locally |
| OpenSCAP | Collects and proxies compliance scan results |

### Capsule Commands

```bash
# See all registered capsules
hammer capsule list

# View details for a specific capsule
hammer capsule info --name capsule.lab.example.com

# Assign a lifecycle environment to a capsule
hammer capsule content add-lifecycle-environment \
  --name capsule.lab.example.com \
  --organization "Operations" \
  --lifecycle-environment "Production"

# Push content to the capsule
hammer capsule content synchronize \
  --name capsule.lab.example.com

# Check sync progress
hammer capsule content sync-status \
  --name capsule.lab.example.com

# See which environments are assigned
hammer capsule content lifecycle-environments \
  --name capsule.lab.example.com
```

### Capsule Sizing Guide

| Managed Hosts | RAM | Disk | CPU |
|---|---|---|---|
| Up to 500 | 12 GB | 500 GB | 4 cores |
| 500 – 2,000 | 16 GB | 1 TB | 8 cores |
| 2,000 – 5,000 | 32 GB | 2 TB | 16 cores |

> **⚠️ Warning:** Always sync Satellite first, then sync Capsules. Never the reverse.

> **⚠️ Warning:** Satellite and Capsule must run the same RHEL major version. Mixed versions cause hard-to-diagnose failures.

---

## 11. Backup & Recovery

### What Gets Backed Up

| Data | Location | Typical Size |
|---|---|---|
| PostgreSQL database | `/var/lib/pgsql/` | 1–10 GB |
| Candlepin (subscriptions) | Included in DB backup | 100–500 MB |
| Configuration files | `/etc/foreman/`, `/etc/pulp/` | 10–50 MB |
| SSL certificates | `/etc/pki/` | 1–5 MB |
| Pulp content (all RPMs) | `/var/lib/pulp/` | 50 GB – 2 TB |

### Backup Commands

```bash
# Full online backup — Satellite keeps running
satellite-maintain backup online \
  --assumeyes \
  /backup/satellite

# Skip RPM content — faster, for lab/testing only
satellite-maintain backup online \
  --assumeyes \
  --skip-pulp-content \
  /backup/satellite

# Incremental backup — only changes since last backup
satellite-maintain backup online \
  --assumeyes \
  --incremental /backup/satellite/PREVIOUS_BACKUP_DIR \
  /backup/satellite

# Offline backup — most consistent, use during maintenance windows
satellite-maintain backup offline \
  --assumeyes \
  /backup/satellite

# Restore from backup (WARNING: this overwrites your current config!)
satellite-maintain restore --assumeyes /backup/satellite/BACKUP_DIR

# Inspect backup metadata
cat /backup/satellite/*/metadata.yml

# Check how much space the backup is using
du -sh /backup/satellite/*/
```

### Recommended Backup Schedule

| Schedule | Type | Notes |
|---|---|---|
| Sunday 1:00 AM | Full backup | Includes all Pulp content |
| Mon–Sat 1:00 AM | Incremental | Only changed data — much faster |
| Monthly | Offsite copy | `rsync` to a remote server |
| Quarterly | Test restore | Restore to a test environment and verify |

> **⚠️ Warning:** A backup you've never tested is not a backup. Restore to a test system every quarter.

> **⚠️ Warning:** In production, always include Pulp content in backups. `--skip-pulp-content` is only acceptable in lab environments.

---

## 12. Daily Admin Checklist & Reference Commands

### Morning Health Check Routine

```bash
# 1. Verify all services are running
satellite-maintain service list

# 2. Run the full health check
satellite-maintain health check

# 3. Check overnight sync results
#    Web UI: Content → Sync Status

# 4. Check for failed jobs
hammer job-invocation list

# 5. Monitor disk usage
df -h /var/lib/pulp /var/lib/pgsql

# 6. Check for paused or stuck tasks
#    Web UI: Monitor → Jobs → filter by "paused"
```

### Quick Reference: Essential Hammer Commands

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

### Troubleshooting Reference

| Symptom | Most Likely Cause | Fix |
|---|---|---|
| HTTP 401 on registration | Satellite CA cert not installed | Install the `katello-ca-consumer` RPM |
| "Activation key not found" | Key is in a different org | Run `hammer activation-key list` per org |
| REX "Permission denied" | SSH key missing on host | Copy `foreman-proxy` public key to host |
| Sync failed | Disk full or CDN unreachable | `df -h /var/lib/pulp`; check firewall |
| Web UI slow / timing out | Stuck tasks or low memory | Run `satellite-maintain health check` |
| Host sees outdated content | Content View not promoted | Run `hammer content-view version promote` |
| Manifest refresh fails | Disconnected/SCA environment | Expected in lab; not an error |
| Services won't start | Disk full or database issue | `df -h`; check `journalctl -u postgresql` |

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
| `utility.lab.example.com` | 172.25.250.220 | DNS, LDAP, and utility services |
| `classroom.example.com` | 172.25.254.254 | Classroom materials server |

### Default Credentials

| System | Username | Password |
|---|---|---|
| All hosts (SSH) | `student` | `student` |
| Root on all hosts | `root` | `redhat` |
| Satellite Web UI | `admin` | `redhat` |
| Satellite CLI (hammer) | `admin` | `redhat` |

### Terminal Prompt Reference

```bash
[student@workstation ~]$   # Your main workstation — start all labs here
[root@satellite ~]#        # Satellite server — run hammer commands here
[root@capsule ~]#          # Capsule server
[root@servera ~]#          # Content host A
[root@serverc ~]#          # Content host C
[root@serverd ~]#          # Content host D
```

> **⚠️ Warning:** Always check your terminal prompt before running commands. `subscription-manager` runs on **content hosts**; `hammer` runs on the **Satellite server**. Running them in the wrong place is a common mistake.

---

## Lessons from the Field

These are the most common mistakes in production — and how to avoid them:

1. **Check your Organization first.** The Organization selector in the top-left of the Web UI controls everything. Working in the wrong org and creating content there is the most common mistake — and it wastes a lot of time to unwind.

2. **Use org labels, not display names.** Labels have no spaces (`Default_Organization`, `Operations`). Display names with spaces break CLI commands and cause 401 errors.

3. **Publishing ≠ Deploying.** Publishing a Content View only creates a snapshot in Library. You must also promote it to a lifecycle environment before any host in that environment can see new content.

4. **Activation keys need three things.** Every activation key must have an Organization, a Lifecycle Environment, and a Content View assigned. Missing any one of these will fail registration with HTTP 500.

5. **REX needs SSH keys.** If you see `Permission denied (publickey)`, the `foreman-proxy` public key isn't on that host yet. Add it manually.

6. **DNS must work in both directions.** Satellite requires both forward (hostname → IP) and reverse (IP → hostname) DNS to work. Broken reverse DNS causes SSL errors and host registration failures.

7. **Watch disk space on `/var/lib/pulp`.** This is where all synced RPMs live. In production it can grow to 500 GB – 2 TB+. When it fills up, syncs fail silently.

8. **Test your restores.** Run a full restore drill on a test system every quarter. Finding out your backup is corrupt during an actual outage is the worst possible time.

---

*Author: Sajal Jana | Red Hat Satellite 6 Administration (RH403) v6.15 | March 2026*
