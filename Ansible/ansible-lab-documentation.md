# Ansible Hands-On Lab — AWS EC2 + RHEL 9
> **Author:** Sajal Jana  
> **Date:** January, 2026  
> **Environment:** AWS EC2 (ap-south-1) · RHEL 9.6 → 9.7 · Ansible Core 2.14.18  
> **Purpose:** Real-world production Ansible lab covering patching, deployment, secrets management and roles

---

## Table of Contents

1. [Lab Architecture](#1-lab-architecture)
2. [Control Node Setup](#2-control-node-setup)
3. [Ansible Installation](#3-ansible-installation)
4. [Project Structure & Configuration](#4-project-structure--configuration)
5. [Inventory Management](#5-inventory-management)
6. [SSH Key Setup](#6-ssh-key-setup)
7. [Ad-hoc Commands & Fact Gathering](#7-ad-hoc-commands--fact-gathering)
8. [First Playbook — Nginx Install](#8-first-playbook--nginx-install)
9. [Idempotency](#9-idempotency)
10. [OS Patching — Rolling Update](#10-os-patching--rolling-update)
11. [Pre/Post Patch Baseline Collection](#11-prepost-patch-baseline-collection)
12. [Application Deployment](#12-application-deployment)
13. [Application Upgrade](#13-application-upgrade)
14. [Ansible Vault — Secrets Management](#14-ansible-vault--secrets-management)
15. [Ansible Roles](#15-ansible-roles)
16. [Final Project Structure](#16-final-project-structure)
17. [Key Concepts Learned](#17-key-concepts-learned)

---

## 1. Lab Architecture

### Infrastructure

| Node | Instance Type | OS | Role |
|------|-------------|-----|------|
| `ansible-control` | t3.medium | RHEL 9 | Ansible Control Node |
| `web01` | t3.small | RHEL 9 | Web Server (Nginx) |
| `app01` | t3.small | RHEL 9 | Application Server |
| `db01` | t3.small | RHEL 9 | Database Server |

### Network Layout

```
AWS VPC (ap-south-1)
└── ansible-control (172.31.4.44)
    ├── web01 (172.31.32.102)  ← SSH port 22
    ├── app01 (172.31.39.8)    ← SSH port 22
    └── db01  (172.31.39.231)  ← SSH port 22
```

### Design Decisions
- Control node is the **only** node with Ansible installed
- Managed nodes need only **Python 3** and **SSH access**
- All SSH traffic stays **within the VPC** (private IPs only)
- Security Groups restrict SSH to control node's private IP only

---

## 2. Control Node Setup

### EC2 Launch Configuration
- **AMI:** RHEL 9 (latest, from AWS Marketplace)
- **Instance type:** t3.medium
- **Key pair:** `mykey200.pem` (downloaded locally)
- **Security Group:** SSH port 22 from personal IP only
- **Storage:** 20 GB gp3

### SSH Connection
```bash
# Fix key permissions (required by SSH)
chmod 400 ~/Downloads/mykey200.pem

# Connect to control node
ssh -i ~/Downloads/mykey200.pem ec2-user@<PUBLIC_IP>
```

---

## 3. Ansible Installation

### Why EPEL on AWS RHEL 9?
AWS RHEL 9 does not include EPEL by default. The standard `epel-release` package is unavailable in base repos, so EPEL must be installed directly from Fedora's project URL.

```bash
# Install EPEL repo (AWS RHEL9 method)
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Install Ansible from EPEL
sudo dnf install -y ansible

# Verify installation
ansible --version
```

### Verified Output
```
ansible [core 2.14.18]
  config file = /etc/ansible/ansible.cfg
  python version = 3.9.25
  jinja version = 3.1.2
  libyaml = True
```

---

## 4. Project Structure & Configuration

### Create Project Directory
```bash
mkdir -p ~/ansible-lab && cd ~/ansible-lab
```

### ansible.cfg — Project-level Configuration
```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory          = ./inventory
remote_user        = ec2-user
private_key_file   = ~/.ssh/mykey200.pem
host_key_checking  = False
forks              = 5
log_path           = ./ansible.log

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root
EOF
```

### Configuration Explained

| Setting | Value | Purpose |
|---------|-------|---------|
| `inventory` | `./inventory` | Look for hosts in local inventory folder |
| `remote_user` | `ec2-user` | Default SSH user for RHEL on AWS |
| `private_key_file` | `~/.ssh/mykey200.pem` | SSH key for managed nodes |
| `host_key_checking` | `False` | Skip SSH fingerprint prompts |
| `forks` | `5` | Parallel connections to hosts |
| `log_path` | `./ansible.log` | Save all run logs locally |
| `become` | `True` | Auto-escalate to root |
| `become_method` | `sudo` | Use sudo for privilege escalation |

> **Production note:** `host_key_checking = False` is acceptable in a controlled VPC lab. Set to `True` in production.

---

## 5. Inventory Management

### Create Inventory Directory
```bash
mkdir -p ~/ansible-lab/inventory
```

### Static Inventory — hosts.ini
```bash
cat > ~/ansible-lab/inventory/hosts << 'EOF'
[webservers]
web01 ansible_host=172.31.32.102

[appservers]
app01 ansible_host=172.31.39.8

[dbservers]
db01 ansible_host=172.31.39.231

[all_servers:children]
webservers
appservers
dbservers

[all_servers:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

### Inventory Concepts

- **Groups** (`[webservers]`, `[appservers]`, `[dbservers]`) — target specific tiers
- **Group of groups** (`[all_servers:children]`) — run tasks across all nodes
- **Group vars** (`[all_servers:vars]`) — apply variables to all hosts in group

### Verify Inventory
```bash
ansible-inventory --list -y       # YAML view of all hosts
ansible all --list-hosts           # flat list of all hosts
ansible webservers --list-hosts    # filter by group
```

---

## 6. SSH Key Setup

### Copy Key to Control Node
The SSH private key must reside on the control node to authenticate against managed nodes.

```bash
# On Windows PowerShell — copy key to control node
scp -i C:\Users\SajalJana\Downloads\mykey200.pem \
    C:\Users\SajalJana\Downloads\mykey200.pem \
    ec2-user@<CONTROL_NODE_IP>:~/.ssh/mykey200.pem

# On control node — set correct permissions
chmod 400 ~/.ssh/mykey200.pem
```

### Test Connectivity to All Nodes
```bash
ansible all -m ping
```

### Expected Output
```
web01 | SUCCESS => { "changed": false, "ping": "pong" }
app01 | SUCCESS => { "changed": false, "ping": "pong" }
db01  | SUCCESS => { "changed": false, "ping": "pong" }
```

> **Note:** Ansible `ping` is not ICMP ping — it tests SSH connectivity and Python availability on the managed node.

---

## 7. Ad-hoc Commands & Fact Gathering

Ad-hoc commands run one-off tasks without writing a playbook. Used for quick checks, emergency fixes and auditing.

### Connectivity Test
```bash
ansible all -m ping
```

### Gather OS Facts
```bash
# Filter specific facts — much cleaner than full setup output
ansible all -m setup -a "filter=ansible_distribution*"
```

**Output across all 3 nodes:**
```json
{
    "ansible_distribution": "RedHat",
    "ansible_distribution_version": "9.6",
    "ansible_distribution_release": "Plow"
}
```

### Check Memory
```bash
ansible all -m setup -a "filter=ansible_memtotal_mb"
```
**Result:** All nodes = 717 MB (t3.small)

### Check Kernel Version
```bash
ansible all -m command -a "uname -r"
```
**Result:** `5.14.0-570.72.1.el9_6.x86_64` (before patching)

### Other Useful Ad-hoc Commands
```bash
# Check disk space
ansible all -m shell -a "df -h /"

# Restart a service
ansible webservers -m service -a "name=nginx state=restarted"

# Copy a file
ansible all -m copy -a "src=./file.conf dest=/etc/app/file.conf mode=0644"

# Check service status
ansible webservers -m command -a "systemctl status nginx"
```

---

## 8. First Playbook — Nginx Install

### Create Playbooks Directory
```bash
mkdir -p ~/ansible-lab/playbooks
```

### Playbook — install-nginx.yml
```yaml
---
- name: Install and Start Nginx on Web Server
  hosts: webservers
  become: true

  tasks:
    - name: Install nginx
      dnf:
        name: nginx
        state: present

    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: true
```

### Dry Run First (Always)
```bash
ansible-playbook playbooks/install-nginx.yml --check
```

### Execute
```bash
ansible-playbook playbooks/install-nginx.yml
```

### Output
```
TASK [Install nginx]          → changed: [web01]
TASK [Start and enable nginx] → changed: [web01]
PLAY RECAP: ok=3  changed=2  failed=0
```

### Verify
```bash
ansible webservers -m command -a "systemctl status nginx"
```
**Result:** `Active: active (running)`

---

## 9. Idempotency

Running the same playbook multiple times must produce the **same result** — no unintended changes.

```bash
# Run the nginx playbook a second time
ansible-playbook playbooks/install-nginx.yml
```

### First Run vs Second Run

| Task | First Run | Second Run |
|------|-----------|------------|
| Install nginx | 🟡 changed | ✅ ok |
| Start nginx | 🟡 changed | ✅ ok |
| Total changes | 2 | **0** |

> **Why this matters in production:** You can safely run the same playbook daily as a compliance check. It only fixes what's broken and touches nothing else.

---

## 10. OS Patching — Rolling Update

### Patching Strategy
- `serial: 1` — patch **one host at a time**
- Pre-task: check disk space, abort if over 85%
- Reboot only if packages were actually updated
- Post-task: verify SSH is back, print new kernel version

### Playbook — patch-servers.yml
```yaml
---
- name: OS Patching - Rolling Update
  hosts: all_servers
  become: true
  serial: 1

  pre_tasks:
    - name: Check disk space before patching
      shell: df / | awk 'NR==2{print $5}' | tr -d '%'
      register: disk_usage
      failed_when: disk_usage.stdout | int > 85

    - name: Print disk usage
      debug:
        msg: "Disk usage on {{ inventory_hostname }} is {{ disk_usage.stdout }}%"

  tasks:
    - name: Update all packages
      dnf:
        name: "*"
        state: latest
        update_cache: true
      register: patch_result

    - name: Reboot if packages were updated
      reboot:
        msg: "Rebooting after patching"
        reboot_timeout: 300
      when: patch_result.changed

  post_tasks:
    - name: Verify SSH is back
      wait_for_connection:
        timeout: 60

    - name: Print kernel version after patch
      command: uname -r
      register: kernel

    - name: Show kernel version
      debug:
        msg: "{{ inventory_hostname }} kernel is now {{ kernel.stdout }}"
```

### Run Patching
```bash
# Dry run first
ansible-playbook playbooks/patch-servers.yml --check

# Real run
ansible-playbook playbooks/patch-servers.yml
```

### Patching Results

| Host | Disk Before | Kernel Before | Kernel After | Status |
|------|-------------|---------------|--------------|--------|
| web01 | 24% | 5.14.0-570.72.1.el9_6 | 5.14.0-611.38.1.el9_7 | ✅ |
| app01 | 24% | 5.14.0-570.72.1.el9_6 | 5.14.0-611.38.1.el9_7 | ✅ |
| db01 | 24% | 5.14.0-570.72.1.el9_6 | 5.14.0-611.38.1.el9_7 | ✅ |

> All 3 servers upgraded from **RHEL 9.6 → 9.7**. Rebooted cleanly, SSH came back on all nodes.

---

## 11. Pre/Post Patch Baseline Collection

Capture system state before and after patching for audit and comparison.

### Playbook — pre-patch-baseline.yml
```yaml
---
- name: Collect Pre-Patch Baseline
  hosts: all_servers
  become: true

  tasks:
    - name: Get disk usage
      command: df -Th
      register: disk

    - name: Get memory info
      command: free -h
      register: memory

    - name: Get fstab
      command: cat /etc/fstab
      register: fstab

    - name: Get kernel version
      command: uname -r
      register: kernel

    - name: Get uptime
      command: uptime
      register: uptime

    - name: Save baseline to control node
      copy:
        content: |
          ========================================
          HOST: {{ inventory_hostname }}
          DATE: {{ ansible_date_time.iso8601 }}
          ========================================

          --- KERNEL ---
          {{ kernel.stdout }}

          --- DISK (df -Th) ---
          {{ disk.stdout }}

          --- MEMORY (free -h) ---
          {{ memory.stdout }}

          --- UPTIME ---
          {{ uptime.stdout }}

          --- /etc/fstab ---
          {{ fstab.stdout }}

        dest: "/home/ec2-user/ansible-lab/baseline_{{ inventory_hostname }}_{{ ansible_date_time.date }}.txt"
      delegate_to: localhost
```

### Run & Verify
```bash
ansible-playbook playbooks/pre-patch-baseline.yml

# Verify files created on control node
ls -lh ~/ansible-lab/baseline_*.txt
```

### Output Files Created
```
baseline_web01_2026-03-09.txt  (1.3K)
baseline_app01_2026-03-09.txt  (1.3K)
baseline_db01_2026-03-09.txt   (1.3K)
```

### Sample Baseline Content (web01)
```
HOST: web01
DATE: 2026-03-09T11:45:57Z

--- KERNEL ---
5.14.0-611.38.1.el9_7.x86_64

--- DISK (df -Th) ---
Filesystem     Type  Size  Used Avail Use%
/dev/nvme0n1p4 xfs   8.8G  3.1G  5.8G  35%

--- MEMORY (free -h) ---
Mem:  716Mi  336Mi  254Mi
Swap: 0B
```

---

## 12. Application Deployment

### Jinja2 Template — files/index.html
```html
<!DOCTYPE html>
<html>
<head><title>My App v{{ app_version }}</title></head>
<body>
  <h1>Hello from {{ inventory_hostname }}!</h1>
  <p>App Version: {{ app_version }}</p>
  <p>Deployed by Ansible on {{ ansible_date_time.date }}</p>
  <p>Status: Running on kernel {{ ansible_kernel }}</p>
</body>
</html>
```

### Playbook — deploy-app.yml
```yaml
---
- name: Deploy Web Application
  hosts: webservers
  become: true
  vars:
    app_version: "1.0"
    app_dir: /usr/share/nginx/html

  tasks:
    - name: Create app directory
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: nginx
        group: nginx
        mode: '0755'

    - name: Deploy index.html from template
      template:
        src: ../files/index.html
        dest: "{{ app_dir }}/index.html"
        owner: nginx
        group: nginx
        mode: '0644'
      notify: Restart nginx

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: true

    - name: Verify app is responding
      uri:
        url: "http://{{ ansible_host }}"
        status_code: 200
      register: result

    - name: Show response status
      debug:
        msg: "App deployed on {{ inventory_hostname }} - HTTP {{ result.status }}"

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### Deploy & Verify
```bash
# Deploy v1.0
ansible-playbook playbooks/deploy-app.yml

# Verify app is live
curl http://172.31.32.102
```

### Live Response
```html
<h1>Hello from web01!</h1>
<p>App Version: 1.0</p>
<p>Deployed by Ansible on 2026-03-09</p>
```

> **Handler behavior:** `Restart nginx` ran at the **end** of the play, not immediately after the template task. Handlers always run after all tasks complete — by design in Ansible.

---

## 13. Application Upgrade

Upgrade app version using the **same playbook** with `-e` (extra vars) — no playbook modification needed.

```bash
# Deploy v2.0 by overriding the version variable at runtime
ansible-playbook playbooks/deploy-app.yml -e "app_version=2.0"
```

### `-e` / `--extra-vars` Explained

`-e` passes variables at runtime, overriding values defined inside the playbook.

**Variable precedence (lowest → highest):**

| Level | Example |
|-------|---------|
| Role defaults | `defaults/main.yml` |
| Inventory vars | `group_vars/` |
| Playbook vars | `vars: app_version: 1.0` |
| Extra vars `-e` | `-e "app_version=2.0"` ← **always wins** |

> **CI/CD use case:** In Jenkins or GitHub Actions, pass the build number dynamically: `-e "app_version=$BUILD_NUMBER"`

### v1.0 vs v2.0 Comparison

| Task | v1.0 | v2.0 |
|------|------|------|
| Create app directory | changed | ok (already exists) |
| Deploy index.html | changed | changed (new content) |
| HTTP response | 200 ✅ | 200 ✅ |

### Verified Live Response (v2.0)
```html
<h1>Hello from web01!</h1>
<p>App Version: 2.0</p>
<p>Deployed by Ansible on 2026-03-09</p>
<p>Status: Running on kernel 5.14.0-611.38.1.el9_7.x86_64</p>
```

---

## 14. Ansible Vault — Secrets Management

Never store passwords in plaintext. Ansible Vault encrypts sensitive data with AES256.

### Create Vault Password File
```bash
echo "VaultPass@2026" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

### Create Encrypted Secrets File
```bash
mkdir -p ~/ansible-lab/inventory/group_vars/all

ansible-vault create ~/ansible-lab/inventory/group_vars/all/vault.yml \
  --vault-password-file ~/.vault_pass
```

**Contents of vault.yml (before encryption):**
```yaml
vault_db_password: "ProdDB@2026"
vault_app_secret:  "AppSecret@2026"
vault_api_key:     "api-key-abc123xyz"
```

**On disk (AES256 encrypted):**
```
$ANSIBLE_VAULT;1.1;AES256
65396566386561356564353163363764...
```

### Public Variables File (safe to commit)
```yaml
# inventory/group_vars/all/vars.yml
db_password: "{{ vault_db_password }}"
app_secret:  "{{ vault_app_secret }}"
api_key:     "{{ vault_api_key }}"
```

### Vault Commands Reference

```bash
# View encrypted file
ansible-vault view vault.yml --vault-password-file ~/.vault_pass

# Edit encrypted file
ansible-vault edit vault.yml --vault-password-file ~/.vault_pass

# Encrypt a single string
ansible-vault encrypt_string 'MySecret' --name 'db_password'

# Run playbook with vault
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass
```

### Test Vault Playbook
```yaml
---
- name: Test Vault Secrets Are Accessible
  hosts: dbservers
  become: true

  tasks:
    - name: Confirm db_password is loaded
      debug:
        msg: "DB password is defined: {{ db_password | length > 0 }}"

    - name: Show first 3 chars only
      debug:
        msg: "DB password starts with: {{ db_password[:3] }}***"
```

### Verified Output
```
msg: "DB password is defined: True"
msg: "DB password starts with: Pro***"
```

> **Important:** `group_vars` must be placed next to the **inventory** folder (not the playbook folder) for Ansible to auto-load variables correctly.

---

## 15. Ansible Roles

Roles package tasks, variables, templates and handlers into **reusable, shareable units**.

### Create Role Scaffold
```bash
mkdir -p ~/ansible-lab/roles
ansible-galaxy init roles/common
```

### Role Directory Structure
```
roles/common/
├── tasks/
│   └── main.yml        ← core task logic
├── handlers/
│   └── main.yml        ← service restart triggers
├── templates/          ← Jinja2 config templates
├── files/              ← static files to copy
├── vars/
│   └── main.yml        ← role constants (not overridable)
├── defaults/
│   └── main.yml        ← overridable default values
└── meta/
    └── main.yml        ← role dependencies
```

### Common Role Tasks — roles/common/tasks/main.yml
```yaml
---
- name: Install common tools
  dnf:
    name:
      - vim
      - curl
      - wget
      - net-tools
      - tree
    state: present

- name: Set timezone to UTC
  timezone:
    name: UTC

- name: Create admin group
  group:
    name: admins
    state: present

- name: Create admin user
  user:
    name: ansibleadmin
    group: admins
    shell: /bin/bash
    create_home: true
    state: present

- name: Print completion message
  debug:
    msg: "Common role applied successfully on {{ inventory_hostname }}"
```

### Master Playbook — site.yml
```yaml
---
- name: Apply common role to all servers
  hosts: all_servers
  become: true
  roles:
    - common
```

### Run
```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

### Results
```
TASK [common : Install common tools]   → changed: [web01, app01, db01]
TASK [common : Set timezone to UTC]    → ok:      [web01, app01, db01]
TASK [common : Create admin group]     → changed: [web01, app01, db01]
TASK [common : Create admin user]      → changed: [web01, app01, db01]
PLAY RECAP: ok=6  changed=3  failed=0  (all 3 hosts)
```

---

## 16. Final Project Structure

```
ansible-lab/
├── ansible.cfg                          ← project config
├── site.yml                             ← master playbook
├── ansible.log                          ← auto-generated run log
├── inventory/
│   ├── hosts                            ← static inventory
│   └── group_vars/
│       └── all/
│           ├── vars.yml                 ← public variables
│           └── vault.yml                ← AES256 encrypted secrets
├── playbooks/
│   ├── install-nginx.yml                ← install & start nginx
│   ├── patch-servers.yml                ← rolling OS patching
│   ├── pre-patch-baseline.yml           ← capture system state
│   ├── deploy-app.yml                   ← app deploy & upgrade
│   └── test-vault.yml                   ← vault secrets test
├── roles/
│   └── common/                          ← reusable base role
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── defaults/main.yml
│       └── vars/main.yml
└── files/
    └── index.html                       ← Jinja2 app template
```

---

## 17. Key Concepts Learned

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Idempotency** | Running playbooks multiple times produces same result — no unintended changes |
| **Inventory** | Defines what hosts Ansible manages, grouped by role/tier |
| **Facts** | System information auto-collected by Ansible (`setup` module) |
| **Handlers** | Tasks that run only when notified — used for service restarts |
| **Serial** | Controls how many hosts are patched simultaneously |
| **`-e` Extra Vars** | Override playbook variables at runtime — key for CI/CD pipelines |
| **Jinja2 Templates** | Dynamic config files with variables resolved at run time |
| **Vault** | AES256 encryption for secrets — safe to commit encrypted files to Git |
| **Roles** | Reusable, structured packages of tasks — foundation of scalable Ansible |
| **`delegate_to`** | Run a task on a different host (e.g., save file to control node) |

### Production Best Practices Applied

- ✅ Always **dry run** (`--check`) before executing playbooks
- ✅ Use `serial: 1` for rolling updates — never patch all hosts simultaneously
- ✅ Pre-task disk checks before patching — abort if disk > 85%
- ✅ Capture baseline **before** changes for audit trail
- ✅ Use `wait_for_connection` to confirm hosts come back after reboot
- ✅ Store secrets in **Ansible Vault**, never in plaintext
- ✅ Separate public `vars.yml` from encrypted `vault.yml`
- ✅ Use `group_vars/all/` for variables that apply to all hosts
- ✅ Use **Roles** for reusable, maintainable automation
- ✅ Use `site.yml` as master entry point for enterprise playbooks
- ✅ Use handlers for service restarts — they run once at end of play
- ✅ Use **symlink pattern** for zero-downtime app deployments

### Modules Used

| Module | Purpose |
|--------|---------|
| `ping` | Test SSH connectivity and Python availability |
| `setup` | Gather system facts |
| `dnf` | Install/remove packages on RHEL |
| `service` | Manage systemd services |
| `file` | Manage files, directories and permissions |
| `copy` | Copy files to managed nodes |
| `template` | Deploy Jinja2 templates |
| `command` | Run commands (no shell features) |
| `shell` | Run shell commands (supports pipes, redirects) |
| `uri` | Make HTTP requests — used for health checks |
| `reboot` | Reboot and wait for host to come back |
| `wait_for_connection` | Wait until host is reachable again |
| `debug` | Print messages and variable values |
| `user` | Manage Linux users |
| `group` | Manage Linux groups |
| `timezone` | Set system timezone |

---

## Environment Summary

| Item | Value |
|------|-------|
| AWS Region | ap-south-1 (Mumbai) |
| OS | Red Hat Enterprise Linux 9 |
| Kernel Before Patching | 5.14.0-570.72.1.el9_6.x86_64 |
| Kernel After Patching | 5.14.0-611.38.1.el9_7.x86_64 |
| Ansible Version | core 2.14.18 |
| Python Version | 3.9.25 |
| Jinja2 Version | 3.1.2 |

---

*Lab completed on March 9, 2026 — hands-on, step by step, one command at a time.*
