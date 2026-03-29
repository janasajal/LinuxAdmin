# Cisco Firepower Management Center (FMC) Deployment on KVM
### With Day0 Configuration — On-Premises Bare Metal (RHEL 9)

---

> **Author:** Sajal Jana  
> **Platform:** On-Premises Bare Metal Server — Red Hat Enterprise Linux 9  
> **Hypervisor:** KVM / libvirt  
> **Document Version:** 1.0  
> **Last Updated:** March 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Prerequisites](#3-prerequisites)
4. [KVM Host Preparation](#4-kvm-host-preparation)
5. [Network Bridge Setup](#5-network-bridge-setup)
6. [Prepare FMC Image](#6-prepare-fmc-image)
7. [Create Day0 Configuration](#7-create-day0-configuration)
8. [Build Day0 ISO](#8-build-day0-iso)
9. [Deploy FMC VM using virt-install](#9-deploy-fmc-vm-using-virt-install)
10. [Monitor Deployment](#10-monitor-deployment)
11. [Post-Deployment Verification](#11-post-deployment-verification)
12. [Troubleshooting](#12-troubleshooting)
13. [FMC Resource Requirements](#13-fmc-resource-requirements)
14. [Quick Reference Cheatsheet](#14-quick-reference-cheatsheet)

---

## 1. Overview

Cisco Firepower Management Center Virtual (FMCv) is a centralized management solution for Cisco Firepower Threat Defense (FTD) devices. This guide covers:

- Preparing a RHEL 9 bare metal server as a KVM hypervisor host
- Creating a **Day0 configuration file** to automate FMCv initial setup (hostname, IP, DNS, NTP, credentials)
- Packaging the Day0 config into an ISO and attaching it to the VM during deployment
- Verifying the deployment and accessing the FMC Web UI

The Day0 approach eliminates manual console-based first-boot configuration, making it suitable for automated and repeatable deployments.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Bare Metal Server (RHEL 9)                    │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                   KVM Hypervisor                         │  │
│   │                                                          │  │
│   │   ┌────────────────────────────────────────────────┐    │  │
│   │   │              FMCv Virtual Machine               │    │  │
│   │   │                                                 │    │  │
│   │   │   ┌──────────────┐    ┌───────────────────┐    │    │  │
│   │   │   │  Day0 ISO    │───►│   FMC Application │    │    │  │
│   │   │   │  (CDROM)     │    │   (qcow2 disk)    │    │    │  │
│   │   │   └──────────────┘    └───────────────────┘    │    │  │
│   │   │                                                 │    │  │
│   │   │   vNIC (virtio) ──► Management Network          │    │  │
│   │   └────────────────────────────────────────────────┘    │  │
│   │                                                          │  │
│   │   Network Bridge (br0) ──► Physical NIC (ens3)          │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Management Network
                    192.168.1.0/24
                              │
                    ┌─────────┴─────────┐
                    │   FMC Web UI      │
                    │  https://192.168.1.100  │
                    └───────────────────┘
```

---

## 3. Prerequisites

### 3.1 Software & Files Required

| Item | Source | Notes |
|---|---|---|
| RHEL 9 Bare Metal Server | Your infrastructure | Already installed |
| FMCv `.qcow2` image | [Cisco CCO (software.cisco.com)](https://software.cisco.com) | Requires Cisco account |
| `genisoimage` package | RHEL 9 repos | For Day0 ISO creation |
| `virt-install` | RHEL 9 repos | VM deployment tool |
| Active RHEL subscription | Red Hat | For `dnf` package access |

### 3.2 Network Information (Example — adjust to your environment)

| Parameter | Value |
|---|---|
| FMC Management IP | `192.168.1.100` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.1.1` |
| Primary DNS | `8.8.8.8` |
| Secondary DNS | `8.8.4.4` |
| NTP Server | `pool.ntp.org` |
| Host Bridge Interface | `br0` |
| Host Physical NIC | `ens3` (verify on your server) |

> **Note:** Replace all IP addresses and interface names with values appropriate for your environment.

---

## 4. KVM Host Preparation

### 4.1 Verify CPU Virtualization Support

```bash
# Check for hardware virtualization support
grep -E 'vmx|svm' /proc/cpuinfo | head -5

# vmx = Intel VT-x
# svm = AMD-V
# If no output, enable virtualization in server BIOS/UEFI
```

### 4.2 Install KVM and Required Packages

```bash
# Install KVM, libvirt, and virtualization tools
dnf install -y \
  qemu-kvm \
  libvirt \
  libvirt-daemon-kvm \
  virt-install \
  virt-viewer \
  bridge-utils \
  genisoimage \
  libguestfs-tools \
  net-tools

# Enable and start libvirt daemon
systemctl enable --now libvirtd

# Verify libvirt is running
systemctl status libvirtd
```

### 4.3 Validate KVM Host Readiness

```bash
# Run host validation checks
virt-host-validate

# Expected output (all should be PASS):
# QEMU: Checking for hardware virtualization     : PASS
# QEMU: Checking if device /dev/kvm exists       : PASS
# QEMU: Checking if device /dev/kvm is accessible: PASS
# QEMU: Checking if device /dev/vhost-net exists : PASS

# Verify KVM kernel modules are loaded
lsmod | grep kvm

# Expected:
# kvm_intel   (or kvm_amd for AMD processors)
# kvm
```

### 4.4 Add User to libvirt Group (Optional — for non-root management)

```bash
# Add your user to the libvirt group
usermod -aG libvirt $(whoami)

# Re-login or apply group change
newgrp libvirt
```

---

## 5. Network Bridge Setup

A Linux bridge allows the FMC VM to connect directly to the management network.

### 5.1 Identify Physical Network Interface

```bash
# List all network interfaces
ip link show

# List interfaces with IP
ip addr show

# Example output — note your management NIC name (e.g., ens3, eth0, enp3s0)
```

### 5.2 Create Bridge Using nmcli (NetworkManager)

```bash
# Step 1: Create the bridge interface br0
nmcli connection add \
  type bridge \
  ifname br0 \
  con-name br0

# Step 2: Set static IP on the bridge
nmcli connection modify br0 \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.method manual

# Step 3: Add physical NIC (ens3) as a bridge slave
# Replace ens3 with your actual NIC name
nmcli connection add \
  type bridge-slave \
  ifname ens3 \
  master br0 \
  con-name br0-slave-ens3

# Step 4: Bring down original NIC connection
nmcli connection down "Wired connection 1"   # adjust name if different

# Step 5: Bring up the bridge
nmcli connection up br0

# Step 6: Verify bridge is up
ip addr show br0
brctl show br0
```

### 5.3 Verify Bridge

```bash
# Check bridge interfaces
brctl show

# Expected output:
# bridge name   bridge id           STP enabled   interfaces
# br0           8000.xxxxxxxxxxxx   yes           ens3

# Confirm bridge has IP
ip addr show br0
```

> **Important:** After creating the bridge, SSH sessions may briefly drop if you are managing over the same interface. Ensure console access (iDRAC/iLO/BMC) is available as a fallback.

---

## 6. Prepare FMC Image

### 6.1 Create Directory Structure

```bash
# Create dedicated directory for FMC deployment files
mkdir -p /var/lib/libvirt/images/fmc
mkdir -p /var/lib/libvirt/images/fmc/day0

cd /var/lib/libvirt/images/fmc
```

### 6.2 Copy FMC qcow2 Image

```bash
# Copy downloaded FMC qcow2 image to the deployment directory
# Adjust the source path to match where you downloaded the image
cp ~/downloads/Cisco_Firepower_Mgmt_Center_Virtual-7.4.1-xxx.qcow2 \
   /var/lib/libvirt/images/fmc/fmcv.qcow2

# Verify image details
qemu-img info /var/lib/libvirt/images/fmc/fmcv.qcow2
```

**Expected output from `qemu-img info`:**

```
image: fmcv.qcow2
file format: qcow2
virtual size: 250 GiB
disk size: 3.5 GiB
```

### 6.3 Check Image Integrity

```bash
# Run integrity check on the qcow2 image
qemu-img check /var/lib/libvirt/images/fmc/fmcv.qcow2

# No errors should be reported
```

---

## 7. Create Day0 Configuration

The Day0 configuration file automates the FMC first-boot setup. Without it, you would need to manually configure FMC through the console during first boot.

### 7.1 Create the Day0 Config File

```bash
cat > /var/lib/libvirt/images/fmc/day0/day0-config << 'EOF'
{
  "EULA": "accept",
  "Hostname": "fmcv",
  "AdminPassword": "Admin@12345",
  "DNS1": "8.8.8.8",
  "DNS2": "8.8.4.4",
  "IPv4Addr": "192.168.1.100",
  "IPv4Mask": "255.255.255.0",
  "IPv4Gw": "192.168.1.1",
  "IPv6Addr": "",
  "IPv6Mask": "",
  "IPv6Gw": "",
  "NTPServer": "pool.ntp.org",
  "SearchDomains": "example.com",
  "ManagementType": "static"
}
EOF
```

> **Security Note:** Replace `Admin@12345` with a strong password following your organization's password policy. The password must meet Cisco's complexity requirements: minimum 8 characters, at least one uppercase, one lowercase, one number, and one special character.

### 7.2 Day0 Parameter Reference

| Parameter | Description | Example |
|---|---|---|
| `EULA` | Accept end-user license agreement | `accept` |
| `Hostname` | FMC VM hostname | `fmcv` |
| `AdminPassword` | Admin user password | `Admin@12345` |
| `DNS1` | Primary DNS server | `8.8.8.8` |
| `DNS2` | Secondary DNS server | `8.8.4.4` |
| `IPv4Addr` | Management IP address | `192.168.1.100` |
| `IPv4Mask` | Subnet mask | `255.255.255.0` |
| `IPv4Gw` | Default gateway | `192.168.1.1` |
| `IPv6Addr` | IPv6 address (optional) | Leave blank if not used |
| `NTPServer` | NTP time server | `pool.ntp.org` |
| `SearchDomains` | DNS search domain | `example.com` |
| `ManagementType` | IP assignment type | `static` or `dhcp` |

### 7.3 Verify Day0 File

```bash
# Verify the file was created correctly
cat /var/lib/libvirt/images/fmc/day0/day0-config

# Check file has no formatting issues
python3 -m json.tool /var/lib/libvirt/images/fmc/day0/day0-config
```

---

## 8. Build Day0 ISO

The Day0 config file must be packaged into an ISO image that will be attached to the FMC VM as a virtual CD-ROM.

```bash
# Create ISO from the day0-config file
genisoimage \
  -r \
  -o /var/lib/libvirt/images/fmc/day0.iso \
  /var/lib/libvirt/images/fmc/day0/day0-config

# Verify ISO was created
ls -lh /var/lib/libvirt/images/fmc/day0.iso
```

**Expected output:**

```
-rw-r--r-- 1 root root 360K Mar 2026 /var/lib/libvirt/images/fmc/day0.iso
```

---

## 9. Deploy FMC VM using virt-install

### 9.1 Set Correct File Permissions

```bash
# Set ownership to qemu user
chown -R qemu:qemu /var/lib/libvirt/images/fmc/

# Set file permissions
chmod 660 /var/lib/libvirt/images/fmc/fmcv.qcow2
chmod 660 /var/lib/libvirt/images/fmc/day0.iso

# Fix SELinux context (critical on RHEL 9)
restorecon -Rv /var/lib/libvirt/images/fmc/

# Verify permissions and SELinux labels
ls -lhZ /var/lib/libvirt/images/fmc/
```

### 9.2 Deploy FMC VM

```bash
virt-install \
  --name fmcv \
  --memory 32768 \
  --vcpus 8 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/fmc/fmcv.qcow2,\
format=qcow2,bus=virtio,cache=none,io=native \
  --disk path=/var/lib/libvirt/images/fmc/day0.iso,\
device=cdrom,bus=sata \
  --os-variant rhel7.0 \
  --network bridge=br0,model=virtio \
  --graphics vnc,listen=0.0.0.0,port=5910 \
  --video vga \
  --boot hd \
  --noautoconsole \
  --autostart
```

### 9.3 virt-install Parameter Explanation

| Parameter | Value | Purpose |
|---|---|---|
| `--name` | `fmcv` | VM name in libvirt |
| `--memory` | `32768` | 32 GB RAM (Cisco minimum requirement) |
| `--vcpus` | `8` | 8 virtual CPUs |
| `--cpu` | `host-passthrough` | Expose full host CPU features to VM |
| `--disk` (1st) | `fmcv.qcow2` | Main OS and data disk |
| `--disk` (2nd) | `day0.iso` | Day0 config attached as CDROM |
| `--os-variant` | `rhel7.0` | FMCv is based on RHEL 7 |
| `--network` | `bridge=br0` | Connect to management bridge |
| `--graphics` | `vnc,port=5910` | VNC console access |
| `--boot` | `hd` | Boot from hard disk |
| `--noautoconsole` | — | Do not auto-attach console |
| `--autostart` | — | Start VM automatically on host reboot |

### 9.4 Confirm VM is Created

```bash
# List all VMs
virsh list --all

# Expected output:
#  Id   Name   State
# ─────────────────────
#  1    fmcv   running
```

---

## 10. Monitor Deployment

FMC takes approximately **15–25 minutes** to complete first boot and apply the Day0 configuration.

### 10.1 Watch VM Status

```bash
# Watch VM status in real time
watch -n 5 virsh list --all
```

### 10.2 Connect via VNC Console

From the KVM host or a remote machine with VNC client:

```bash
# From KVM host — get VNC display info
virsh vncdisplay fmcv

# Connect using vncviewer from remote machine
vncviewer <KVM-HOST-IP>:5910
```

You will see the Linux boot sequence followed by FMC initialization messages.

### 10.3 Monitor via Serial Console

```bash
# Attach to VM serial console
virsh console fmcv

# Press Ctrl+] to detach from console
```

### 10.4 Watch libvirt Logs

```bash
# Monitor libvirt QEMU logs for fmcv in real time
tail -f /var/log/libvirt/qemu/fmcv.log
```

### 10.5 Expected Boot Sequence

```
Phase 1 — BIOS POST           (~1 min)
Phase 2 — Linux Kernel Boot   (~2 min)
Phase 3 — Day0 ISO Detection  (~1 min)  ← Day0 config is read here
Phase 4 — Applying Day0 Config (~3 min) ← IP, hostname, password set
Phase 5 — FMC Services Start  (~15 min) ← Database, web server, etc.
Phase 6 — FMC Ready           ← Web UI accessible
```

---

## 11. Post-Deployment Verification

### 11.1 Get VM Network Information

```bash
# Get FMC VM IP address
virsh domifaddr fmcv

# Check VM details
virsh dominfo fmcv
```

### 11.2 Test Network Connectivity

```bash
# Ping FMC from KVM host
ping -c 4 192.168.1.100

# Test SSH connectivity
ssh admin@192.168.1.100
```

### 11.3 Verify from FMC CLI

Once SSH is established:

```bash
# Check FMC version
> show version

# Check network configuration
> show network

# Check FMC managers (should show "None" if no FTD registered yet)
> show managers

# Check system status
> show process
```

### 11.4 Access FMC Web UI

1. Open a web browser
2. Navigate to: `https://192.168.1.100`
3. Accept the SSL certificate warning
4. Login with:
   - **Username:** `admin`
   - **Password:** `Admin@12345` *(or the password set in Day0 config)*
5. Complete the initial setup wizard

### 11.5 Verify Day0 Config Was Applied

```bash
# From FMC CLI — verify hostname
> show hostname

# Verify DNS
> show dns

# Verify NTP
> show ntp
```

---

## 12. Troubleshooting

### 12.1 VM Fails to Start

**Symptom:** `virsh start fmcv` returns an error or VM immediately stops.

```bash
# Check libvirt QEMU log
cat /var/log/libvirt/qemu/fmcv.log

# Check system journal for libvirt errors
journalctl -u libvirtd --since "30 minutes ago"

# Verify qcow2 image is not corrupted
qemu-img check /var/lib/libvirt/images/fmc/fmcv.qcow2

# Check available disk space on host
df -h /var/lib/libvirt/images/
```

---

### 12.2 Day0 Configuration Not Applied

**Symptom:** FMC boots but uses default settings — no IP, default hostname.

```bash
# Step 1: Verify ISO is attached to VM
virsh domblklist fmcv

# Expected output:
# Target   Source
# ─────────────────────────────────────────────
# vda      /var/lib/libvirt/images/fmc/fmcv.qcow2
# sda      /var/lib/libvirt/images/fmc/day0.iso

# Step 2: Verify ISO content is correct
mount -o loop /var/lib/libvirt/images/fmc/day0.iso /mnt
cat /mnt/day0-config
umount /mnt

# Step 3: Re-attach ISO and reboot VM
virsh attach-disk fmcv \
  /var/lib/libvirt/images/fmc/day0.iso \
  sda \
  --type cdrom \
  --mode readonly \
  --live

virsh reboot fmcv
```

---

### 12.3 SELinux Blocking VM Access to Image

**Symptom:** VM fails with permission denied errors on RHEL 9.

```bash
# Check for SELinux denials related to libvirt
ausearch -m avc -ts recent | grep libvirt

# Fix SELinux context on image files
restorecon -Rv /var/lib/libvirt/images/

# Verify correct SELinux type (should be svirt_image_t)
ls -lZ /var/lib/libvirt/images/fmc/

# If still failing, temporarily set SELinux to permissive for testing only
setenforce 0
virsh start fmcv
# If VM starts, the issue is confirmed as SELinux
# Fix context then re-enable enforcing mode
setenforce 1
```

---

### 12.4 Network Not Reachable After Boot

**Symptom:** FMC VM boots successfully but cannot be pinged from the network.

```bash
# Step 1: Verify bridge is up and connected to physical NIC
brctl show br0
ip addr show br0

# Step 2: Check VM's virtual NIC
virsh domiflist fmcv

# Step 3: Check VM MAC address and ARP table
virsh domifaddr fmcv
arp -n | grep <fmcv-mac-address>

# Step 4: Check if bridge is carrying traffic
tcpdump -i br0 -n host 192.168.1.100

# Step 5: Check firewalld is not blocking traffic
firewall-cmd --list-all
firewall-cmd --zone=trusted --add-interface=br0 --permanent
firewall-cmd --reload
```

---

### 12.5 Insufficient Host Resources

**Symptom:** VM performance is poor or fails to start due to resource limits.

```bash
# Check available RAM on host
free -h

# Check available CPU cores
lscpu | grep -E 'CPU\(s\)|Thread|Core'

# Check disk space
df -h /var/lib/libvirt/images/

# Check if hugepages are causing memory allocation issues
cat /proc/meminfo | grep -i huge

# Check overcommit settings
cat /proc/sys/vm/overcommit_memory
```

---

### 12.6 VNC Console Not Accessible

**Symptom:** Cannot connect to VM console via VNC.

```bash
# Verify VNC port being used by fmcv
virsh vncdisplay fmcv

# Check if VNC port is listening
ss -tlnp | grep 5910

# Check firewalld — allow VNC port if needed
firewall-cmd --add-port=5910/tcp --permanent
firewall-cmd --reload

# Use virsh console as alternative to VNC
virsh console fmcv
```

---

### 12.7 Web UI Not Accessible

**Symptom:** FMC CLI is reachable but web UI at `https://192.168.1.100` does not load.

```bash
# Check if HTTPS port 443 is open on FMC
# From KVM host:
curl -kv https://192.168.1.100

# From FMC CLI — check web server process
> show process | include httpd

# Check FMC service status via CLI
> show version
> pmtool status

# FMC web services can take up to 25 minutes after first boot
# Wait and retry if services are still initializing
```

---

### 12.8 Common Error Reference

| Error Message | Likely Cause | Fix |
|---|---|---|
| `error: Failed to connect socket to '/var/run/libvirt/libvirt-sock'` | libvirtd not running | `systemctl start libvirtd` |
| `error: Cannot access storage file: Permission denied` | Wrong ownership or SELinux | `chown qemu:qemu` + `restorecon` |
| `error: internal error: process exited while connecting to monitor` | Corrupt image or missing KVM | Check `qemu-img check`, verify `lsmod \| grep kvm` |
| `Could not open backing file` | qcow2 backing chain broken | Use `qemu-img rebase` or re-download image |
| `error: Network not found: no network with matching name 'br0'` | Bridge not defined in libvirt | Create bridge manually or define in libvirt |
| Day0 not applied after reboot | ISO not attached or wrong filename | Verify with `virsh domblklist fmcv` |

---

## 13. FMC Resource Requirements

### Cisco Official Minimum Requirements

| Resource | Minimum | Recommended | Notes |
|---|---|---|---|
| **vCPUs** | 4 | 8 | Host physical CPUs preferred |
| **RAM** | 28 GB | 32 GB | FMC is memory-intensive |
| **Disk (OS)** | 250 GB | 500 GB | qcow2 thin provisioned |
| **NICs** | 1 | 1+ | Management only required |
| **KVM/QEMU** | 2.x+ | Latest stable | RHEL 9 ships latest |
| **RHEL Version** | 8+ | 9 | For host OS |

### FMCv Model Sizing Guide

| FMCv Model | Max Managed Devices | vCPUs | RAM |
|---|---|---|---|
| FMCv2 | 2 | 4 | 28 GB |
| FMCv10 | 10 | 4 | 28 GB |
| FMCv25 | 25 | 8 | 32 GB |
| FMCv300 | 300 | 8 | 32 GB |

> Refer to [Cisco FMCv Data Sheet](https://www.cisco.com/c/en/us/products/security/firepower-management-center/datasheet-listing.html) for the latest sizing details.

---

## 14. Quick Reference Cheatsheet

### Deployment Commands Summary

```bash
# 1. Install packages
dnf install -y qemu-kvm libvirt virt-install genisoimage bridge-utils

# 2. Start libvirt
systemctl enable --now libvirtd

# 3. Create directories
mkdir -p /var/lib/libvirt/images/fmc/day0

# 4. Copy FMC image
cp ~/downloads/fmcv.qcow2 /var/lib/libvirt/images/fmc/fmcv.qcow2

# 5. Create Day0 config (edit IP/password as needed)
cat > /var/lib/libvirt/images/fmc/day0/day0-config << 'EOF'
{"EULA":"accept","Hostname":"fmcv","AdminPassword":"Admin@12345",
 "DNS1":"8.8.8.8","DNS2":"8.8.4.4","IPv4Addr":"192.168.1.100",
 "IPv4Mask":"255.255.255.0","IPv4Gw":"192.168.1.1",
 "NTPServer":"pool.ntp.org","SearchDomains":"example.com","ManagementType":"static"}
EOF

# 6. Build ISO
genisoimage -r -o /var/lib/libvirt/images/fmc/day0.iso \
  /var/lib/libvirt/images/fmc/day0/day0-config

# 7. Fix permissions
chown -R qemu:qemu /var/lib/libvirt/images/fmc/
restorecon -Rv /var/lib/libvirt/images/fmc/

# 8. Deploy VM
virt-install --name fmcv --memory 32768 --vcpus 8 --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/fmc/fmcv.qcow2,format=qcow2,bus=virtio,cache=none,io=native \
  --disk path=/var/lib/libvirt/images/fmc/day0.iso,device=cdrom,bus=sata \
  --os-variant rhel7.0 --network bridge=br0,model=virtio \
  --graphics vnc,listen=0.0.0.0,port=5910 --video vga \
  --boot hd --noautoconsole --autostart
```

### Day-2 VM Management Commands

```bash
# Start / Stop / Restart VM
virsh start fmcv
virsh shutdown fmcv        # Graceful shutdown
virsh destroy fmcv         # Force power off
virsh reboot fmcv

# VM status and info
virsh list --all
virsh dominfo fmcv
virsh domifaddr fmcv
virsh domblklist fmcv

# Console access
virsh console fmcv         # Serial console (Ctrl+] to exit)

# Take a snapshot
virsh snapshot-create-as fmcv fmcv-snap-01 "Before upgrade"

# List snapshots
virsh snapshot-list fmcv

# View VM XML
virsh dumpxml fmcv

# Remove VM (does not delete disk)
virsh undefine fmcv

# Remove VM and delete disk
virsh undefine fmcv --remove-all-storage
```

---

*Document prepared by **Sajal Jana** | On-Premises KVM Deployment Series*  
*For issues or improvements, please open a GitHub issue or pull request.*
