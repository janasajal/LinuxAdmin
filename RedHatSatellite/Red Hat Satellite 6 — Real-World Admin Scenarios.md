# 🛰️ Red Hat Satellite 6 — Real-World Admin Scenarios

> **Author:** Sajal Jana
> **Topic:** Production Troubleshooting & Best Practices Guide
> **Based on:** RH403 v6.15

---

## Table of Contents

1. [Scenario 1 — Emergency Security Patch (CVE Vulnerability)](#scenario-1--emergency-security-patch--cve-vulnerability)
2. [Scenario 2 — Accidental Content View Deletion](#scenario-2--accidental-content-view-deletion)
3. [Scenario 3 — Host Registration Failure (401 Unauthorized)](#scenario-3--host-registration-failure--401-unauthorized)
4. [Scenario 4 — Content Sync Failure (Repos Not Updating)](#scenario-4--content-sync-failure--repos-not-updating)
5. [Scenario 5 — Remote Execution Failure (Permission Denied)](#scenario-5--remote-execution-failure--permission-denied)
6. [Scenario 6 — Manifest Expired (Content Access Lost)](#scenario-6--manifest-expired--content-access-lost)
7. [Scenario 7 — New Server Onboarding (Full Setup)](#scenario-7--new-server-onboarding--full-setup)
8. [Scenario 8 — Satellite Performance Degradation](#scenario-8--satellite-performance-degradation)
9. [Daily Admin Checklist](#daily-admin-checklist)
10. [Hammer Quick Reference](#hammer-quick-reference)

---

## Scenario 1 — Emergency Security Patch — CVE Vulnerability

**Severity:** 🔴 Critical

### The Situation

Monday 8AM. Security team sends urgent message: all RHEL servers are vulnerable to a critical kernel CVE. Patch ALL servers TODAY.

### Your Action Plan

1. Check which hosts are affected
2. Identify the security errata
3. Apply patches to all hosts simultaneously
4. Verify patches applied successfully
5. Report back to security team

### Commands to Execute

**Step 1 — Find all security errata on hosts**

```bash
hammer host errata list \
  --host serverc.lab.example.com
```

**Step 2 — Apply security errata to ALL hosts at once**

```bash
hammer job-invocation create \
  --feature katello_errata_install \
  --search-query "organization = Operations" \
  --inputs errata=security
```

**Step 3 — Monitor the job**

```bash
hammer job-invocation list

hammer job-invocation output \
  --id <JOB_ID> \
  --host serverc.lab.example.com
```

**Step 4 — Verify patches applied**

```bash
hammer host errata list \
  --host serverc.lab.example.com
```

### Expected Result

Security errata count = `0` on all hosts after successful patching.

> 💡 **TIP:** Always patch Dev first, then QA, then Production — even for emergency patches!

> ⚠️ **WARN:** Some patches require a reboot. Check with: `needs-restarting -r`

---

## Scenario 2 — Accidental Content View Deletion

**Severity:** 🔴 Critical

### The Situation

Junior admin accidentally deleted the production Content View. 500 servers can't get updates. Fix it ASAP!

### Your Action Plan

1. Assess what was deleted
2. Check if backup exists
3. Recreate the Content View
4. Add repositories back
5. Publish and promote through pipeline
6. Implement RBAC to prevent recurrence

### Commands to Execute

**Step 1 — Check current content views**

```bash
hammer content-view list \
  --organization "Operations"
```

**Step 2 — Recreate the Content View**

```bash
hammer content-view create \
  --name "OperationsServerBase" \
  --organization "Operations" \
  --description "Recreated Base CV"
```

**Step 3 — Add repositories back**

```bash
hammer content-view add-repository \
  --name "OperationsServerBase" \
  --organization "Operations" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --repository "Red Hat Enterprise Linux 9 for x86_64 - BaseOS RPMs 9"
```

**Step 4 — Publish new version**

```bash
hammer content-view publish \
  --name "OperationsServerBase" \
  --organization "Operations" \
  --description "Emergency recreation"
```

**Step 5 — Promote through pipeline**

```bash
hammer content-view version promote \
  --content-view "OperationsServerBase" \
  --version 1.0 \
  --to-lifecycle-environment "Production" \
  --organization "Operations"
```

### Prevention

- Implement RBAC — junior admins should **NOT** have delete permissions
- Use `hammer role create` with restricted permissions
- Take regular backups with `satellite-maintain backup`

> 💡 **TIP:** Always run `satellite-maintain backup` before any major changes!

---

## Scenario 3 — Host Registration Failure — 401 Unauthorized

**Severity:** 🟠 High

### The Situation

New RHEL server fails to register to Satellite with HTTP 401 error. Server can't receive updates or be managed.

### Root Causes & Fixes

| Root Cause | Fix |
|---|---|
| Wrong Satellite URL | Install `katello-ca-consumer` RPM |
| Wrong organization name | Use label not name (`hammer org list`) |
| Wrong activation key | Verify with `hammer activation-key list` |
| SSL certificate missing | Re-install `katello-ca-consumer` RPM |
| Activation key in wrong org | Check all orgs for the key |

### Diagnostic Commands

**Step 1 — Install Satellite CA certificate**

```bash
curl -k https://satellite.lab.example.com/pub/katello-ca-consumer-latest.noarch.rpm \
  -o /tmp/katello-ca.rpm

rpm -Uvh /tmp/katello-ca.rpm
```

**Step 2 — Find correct org label**

```bash
hammer organization list
```

**Step 3 — Find activation key**

```bash
hammer activation-key list --organization "Operations"
```

**Step 4 — Register with correct details**

```bash
subscription-manager register \
  --org Operations \
  --activationkey OperationsServers
```

> 💡 **TIP:** Always use org **LABEL** not name. Labels never have spaces!

> ⚠️ **WARN:** One activation key MUST have a lifecycle environment and content view assigned!

---

## Scenario 4 — Content Sync Failure — Repos Not Updating

**Severity:** 🟠 High

### The Situation

Daily sync job failed overnight. Satellite repos are outdated. Hosts are not getting latest patches.

### Diagnostic Steps

**Step 1 — Check sync status**

```bash
# Web UI: Content → Sync Status
hammer repository list \
  --organization "Default Organization"
```

**Step 2 — Check Satellite health**

```bash
satellite-maintain health check
```

**Step 3 — Check Satellite services**

```bash
satellite-maintain service list
satellite-maintain service status
```

**Step 4 — Check disk space (most common cause!)**

```bash
df -h /var/lib/pulp
df -h /var/lib/pgsql
```

**Step 5 — Manually trigger sync**

```bash
hammer product synchronize \
  --name "Red Hat Enterprise Linux for x86_64" \
  --organization "Default Organization"
```

**Step 6 — Check sync plan is enabled**

```bash
hammer sync-plan info \
  --name "Daily Sync" \
  --organization "Default Organization"
```

### Common Fixes

| Problem | Fix |
|---|---|
| Disk full on `/var/lib/pulp` | Delete old Content View versions |
| Pulp service down | `systemctl restart pulpcore` |
| CDN unreachable | Check firewall and DNS |
| Sync plan disabled | `hammer sync-plan update --enabled true` |

> ⚠️ **WARN:** `/var/lib/pulp` needs 300 GB+ in production. Monitor disk space daily!

---

## Scenario 5 — Remote Execution Failure — Permission Denied

**Severity:** 🟡 Medium

### The Situation

REX jobs failing with `Permission denied (publickey)` error. Can't run remote commands on managed hosts.

### Root Cause

Satellite's REX SSH public key is not in the target host's `authorized_keys` file.

### Fix — Distribute SSH Key Manually

**Step 1 — Get Satellite REX public key**

```bash
cat /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub
```

**Step 2 — Add key to target host**

```bash
# Run on the target host
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo "<PASTE_KEY_HERE>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

**Step 3 — Test SSH connection from Satellite**

```bash
ssh -i /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy \
  root@serverc.lab.example.com 'hostname'
```

**Step 4 — Rerun the job**

```bash
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="uptime" \
  --search-query "name = serverc.lab.example.com"
```

> 💡 **TIP:** In production, use REX Pull mode to avoid SSH key distribution issues at scale!

---

## Scenario 6 — Manifest Expired — Content Access Lost

**Severity:** 🔴 Critical

### The Situation

Red Hat subscription manifest has expired. Satellite can't sync new content. Hosts showing subscription warnings.

### Check Manifest Expiry

```bash
hammer subscription list \
  --organization "Default Organization" \
  --fields "Name,End Date,Quantity"
```

### Fix — Renew and Refresh Manifest

1. Log into `access.redhat.com`
2. Navigate to **Subscriptions → Subscription Allocations**
3. Renew subscriptions for your Satellite allocation
4. Download new manifest zip file
5. Upload to Satellite: **Content → Subscriptions → Manage Manifest**

**Or refresh via hammer (after renewing on the portal)**

```bash
hammer subscription refresh-manifest \
  --organization "Default Organization"
```

### Prevention

- Set a calendar reminder **60 days** before expiry
- Monitor expiry monthly with `hammer subscription list`
- Configure email alerts in Satellite for expiring subscriptions

> ⚠️ **WARN:** Expired manifest = no new content syncs = unpatched servers!

---

## Scenario 7 — New Server Onboarding — Full Setup

**Severity:** 🟡 Medium

### The Situation

New RHEL 9 server added to datacenter. Needs to be fully configured and managed by Satellite.

### Complete Onboarding Checklist

| Step | Action | Command / Location |
|---|---|---|
| 1 | Install Katello CA | `dnf localinstall http://satellite/pub/katello-ca-consumer-latest.noarch.rpm` |
| 2 | Register to Satellite | `subscription-manager register --org Operations --activationkey OperationsServers` |
| 3 | Verify registration | `subscription-manager status` |
| 4 | Install katello-agent | `dnf install katello-agent -y` |
| 5 | Apply all errata | `hammer job-invocation create --feature katello_errata_install` |
| 6 | Verify in Web UI | Hosts → Content Hosts → verify host appears |
| 7 | Add to host collection | `hammer host-collection add-host --name OpsServers` |
| 8 | Test REX | `hammer job-invocation create --inputs command=hostname` |

> 💡 **TIP:** Create a separate activation key per environment (Dev, QA, Prod) for clean separation!

---

## Scenario 8 — Satellite Performance Degradation

**Severity:** 🟠 High

### The Situation

Satellite Web UI is slow. Jobs taking too long. Admins complaining about timeouts.

### Diagnostic Commands

**Step 1 — Check all services**

```bash
satellite-maintain health check
```

**Step 2 — Check for stuck/paused tasks**

```bash
# Web UI: Monitor → Jobs → filter by paused
hammer task list --search "state=paused"
```

**Step 3 — Check disk and memory**

```bash
df -h
free -g
top
```

**Step 4 — Check PostgreSQL**

```bash
satellite-maintain service status --only postgresql
```

**Step 5 — Restart services if needed**

```bash
satellite-maintain service restart
```

### Common Performance Fixes

| Problem | Fix |
|---|---|
| Stuck/paused tasks | Monitor → Jobs → select all paused → Discard |
| Bloated PostgreSQL | `foreman-rake db:vacuum` |
| Too many facts stored | Administer → Settings → `facts_clean_up` |
| Disk full | Delete old Content View versions |
| High RAM usage | Increase server RAM above 20 GB |

---

## Daily Admin Checklist

Run these checks every morning to keep Satellite healthy:

| Morning Check | Command / Location |
|---|---|
| All services running | `satellite-maintain service list` |
| Run health check | `satellite-maintain health check` |
| Overnight sync status | Web UI: Content → Sync Status |
| Failed jobs | `hammer job-invocation list` |
| Security errata | Web UI: Monitor → Dashboard |
| Disk space | `df -h /var/lib/pulp /var/lib/pgsql` |
| Paused tasks | Web UI: Monitor → Jobs → filter paused |

---

## Hammer Quick Reference

| Task | Command |
|---|---|
| List all hosts | `hammer host list --organization "Org"` |
| List content views | `hammer content-view list --organization "Org"` |
| List activation keys | `hammer activation-key list --organization "Org"` |
| List lifecycle envs | `hammer lifecycle-environment list --organization "Org"` |
| Check host errata | `hammer host errata list --host hostname` |
| Apply security patches | `hammer job-invocation create --feature katello_errata_install` |
| Run remote command | `hammer job-invocation create --job-template "Run Command - Script Default"` |
| Check job status | `hammer job-invocation list` |
| Check capsule status | `hammer capsule list` |
| Backup Satellite | `satellite-maintain backup online --assumeyes /backup/satellite` |
| Health check | `satellite-maintain health check` |
| Service status | `satellite-maintain service list` |

---

*Author: Sajal Jana | Red Hat Satellite 6 Administration (RH403) v6.15 | March 2026*
