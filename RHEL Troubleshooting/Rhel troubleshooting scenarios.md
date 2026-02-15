# RHEL Troubleshooting Guide: Top 20 Real-World Scenarios

**Author:** Sajal Jana  
**Focus:** Red Hat Enterprise Linux (RHEL) Production Environment Troubleshooting

---

## Table of Contents
1. [High CPU Usage by Unknown Process](#scenario-1-high-cpu-usage-by-unknown-process)
2. [System Running Out of Memory](#scenario-2-system-running-out-of-memory)
3. [Disk Space Full on Root Partition](#scenario-3-disk-space-full-on-root-partition)
4. [Network Connectivity Issues](#scenario-4-network-connectivity-issues)
5. [Service Fails to Start After Reboot](#scenario-5-service-fails-to-start-after-reboot)
6. [SSH Connection Timeout](#scenario-6-ssh-connection-timeout)
7. [System Boot Failure - Kernel Panic](#scenario-7-system-boot-failure---kernel-panic)
8. [High Load Average but Low CPU Usage](#scenario-8-high-load-average-but-low-cpu-usage)
9. [File System Corruption](#scenario-9-file-system-corruption)
10. [SELinux Blocking Application](#scenario-10-selinux-blocking-application)
11. [NFS Mount Issues](#scenario-11-nfs-mount-issues)
12. [Zombie Processes Accumulation](#scenario-12-zombie-processes-accumulation)
13. [DNS Resolution Failure](#scenario-13-dns-resolution-failure)
14. [Time Synchronization Problems](#scenario-14-time-synchronization-problems)
15. [RPM Database Corruption](#scenario-15-rpm-database-corruption)
16. [Firewall Blocking Legitimate Traffic](#scenario-16-firewall-blocking-legitimate-traffic)
17. [LVM Volume Group Issues](#scenario-17-lvm-volume-group-issues)
18. [Performance Degradation After Patching](#scenario-18-performance-degradation-after-patching)
19. [User Unable to Login](#scenario-19-user-unable-to-login)
20. [Journal/Log Files Consuming Excessive Space](#scenario-20-journallog-files-consuming-excessive-space)

---

## Scenario 1: High CPU Usage by Unknown Process

### Problem
Server experiencing 100% CPU utilization, applications responding slowly.

### Diagnosis Steps
```bash
# Check current CPU usage
top -bn1 | head -20

# Identify top CPU consumers
ps aux --sort=-%cpu | head -10

# Check for specific user processes
ps -u username -o pid,ppid,%cpu,%mem,cmd --sort=-%cpu

# Detailed process investigation
lsof -p <PID>
cat /proc/<PID>/cmdline
```

### Root Cause
Runaway cron job executing an infinite loop script.

### Solution
```bash
# Kill the problematic process
kill -15 <PID>  # Graceful termination
kill -9 <PID>   # Force kill if necessary

# Check cron configuration
crontab -l -u username
vi /etc/crontab

# Fix the script logic
# Add proper exit conditions and error handling
```

### Prevention
- Implement timeouts in cron jobs
- Add resource limits using ulimit
- Monitor cron job execution times
- Use `timeout` command for scripts

---

## Scenario 2: System Running Out of Memory

### Problem
System becomes unresponsive, OOM killer terminating processes.

### Diagnosis Steps
```bash
# Check memory usage
free -h
vmstat 1 5

# Identify memory-intensive processes
ps aux --sort=-%mem | head -10

# Check OOM killer logs
grep -i "out of memory" /var/log/messages
dmesg | grep -i "killed process"

# Analyze swap usage
swapon -s
cat /proc/swaps
```

### Root Cause
Memory leak in Java application combined with insufficient swap space.

### Solution
```bash
# Immediate: Restart the problematic service
systemctl restart java-app.service

# Add swap space
dd if=/dev/zero of=/swapfile bs=1G count=4
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Tune JVM parameters
vi /etc/systemd/system/java-app.service
# Add: -Xmx2g -Xms2g parameters

# Configure OOM score
echo -1000 > /proc/<critical_process_PID>/oom_score_adj
```

### Prevention
- Set appropriate JVM heap sizes
- Monitor memory trends with monitoring tools
- Implement application-level memory profiling
- Configure proper swap space (2x RAM for systems <2GB, 1x RAM for larger)

---

## Scenario 3: Disk Space Full on Root Partition

### Problem
Root partition at 100%, system unable to write logs or create temporary files.

### Diagnosis Steps
```bash
# Check disk usage
df -h
df -i  # Check inode usage

# Find large files
du -sh /* | sort -rh | head -10
find / -type f -size +100M -exec ls -lh {} \;

# Check for deleted files still held by processes
lsof +L1

# Identify largest directories
du -hx --max-depth=2 / | sort -rh | head -20
```

### Root Cause
Log rotation not configured properly, old log files accumulating in /var/log.

### Solution
```bash
# Immediate cleanup
# Compress old logs
gzip /var/log/*.log

# Remove old compressed logs
find /var/log -name "*.gz" -mtime +30 -delete

# Clear journal logs
journalctl --vacuum-time=7d
journalctl --vacuum-size=500M

# Configure logrotate
vi /etc/logrotate.d/application
```

Example logrotate configuration:
```
/var/log/application/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 appuser appgroup
}
```

### Prevention
- Implement comprehensive log rotation
- Set up disk space monitoring alerts (80% threshold)
- Use separate partitions for /var, /tmp, /home
- Configure log retention policies

---

## Scenario 4: Network Connectivity Issues

### Problem
Server cannot communicate with external networks, intermittent connection drops.

### Diagnosis Steps
```bash
# Check network interface status
ip addr show
ip link show
nmcli device status

# Test connectivity
ping -c 4 8.8.8.8
ping -c 4 google.com

# Check routing
ip route show
traceroute google.com

# DNS resolution test
nslookup google.com
dig google.com

# Check network statistics
netstat -i
ip -s link

# Check for packet loss
mtr -r -c 100 google.com
```

### Root Cause
NetworkManager conflicting with network scripts, incorrect DNS configuration.

### Solution
```bash
# Disable NetworkManager if using network scripts
systemctl stop NetworkManager
systemctl disable NetworkManager

# Configure static network
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

Example configuration:
```
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```

```bash
# Restart network
systemctl restart network

# Or use nmcli
nmcli connection modify eth0 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection up eth0
```

### Prevention
- Document network configuration standards
- Use configuration management tools (Ansible)
- Implement redundant network paths
- Monitor network metrics continuously

---

## Scenario 5: Service Fails to Start After Reboot

### Problem
Critical application service doesn't start automatically after system reboot.

### Diagnosis Steps
```bash
# Check service status
systemctl status application.service

# View service logs
journalctl -u application.service -n 50

# Check service configuration
systemctl cat application.service

# Verify service dependencies
systemctl list-dependencies application.service

# Check if service is enabled
systemctl is-enabled application.service
```

### Root Cause
Service dependency on network mount not configured, service starting before NFS mount available.

### Solution
```bash
# Edit service file
systemctl edit application.service
```

Add dependency:
```
[Unit]
After=network-online.target remote-fs.target
Wants=network-online.target
Requires=remote-fs.target

[Service]
RestartSec=10
Restart=on-failure
```

```bash
# Reload systemd
systemctl daemon-reload

# Enable service
systemctl enable application.service

# Test restart
systemctl restart application.service
```

### Prevention
- Properly define service dependencies
- Test services across reboots
- Use `After=` and `Requires=` directives appropriately
- Implement health checks

---

## Scenario 6: SSH Connection Timeout

### Problem
Unable to SSH into the server, connection times out.

### Diagnosis Steps
```bash
# From another server, check if SSH port is open
telnet server_ip 22
nc -zv server_ip 22

# On the affected server (via console):
# Check SSH service
systemctl status sshd

# Check SSH configuration
sshd -t
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"

# Check firewall rules
firewall-cmd --list-all
iptables -L -n -v

# Check SELinux
getenforce
ausearch -m avc -ts recent

# Check network interface
ip addr show
```

### Root Cause
Firewall rules blocking SSH after security update, SSH daemon not listening on correct interface.

### Solution
```bash
# Add firewall rule
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# Or for specific IP
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="ssh" accept'
firewall-cmd --reload

# Configure SSH to listen on all interfaces
vi /etc/ssh/sshd_config
# Ensure: ListenAddress 0.0.0.0

# Restart SSH
systemctl restart sshd
```

### Prevention
- Document firewall changes
- Use configuration management
- Implement SSH key-based authentication
- Set up backup access methods (console, serial)

---

## Scenario 7: System Boot Failure - Kernel Panic

### Problem
System fails to boot, showing kernel panic error.

### Diagnosis Steps
```bash
# Boot into rescue mode (from GRUB menu):
# - Select older kernel version
# - Add 'rescue' or 'emergency' to kernel parameters

# Once in rescue mode:
# Check system logs
journalctl -xb
dmesg | less

# Check filesystem integrity
fsck -y /dev/sda1

# Review recent kernel updates
rpm -qa | grep kernel
```

### Root Cause
Corrupted initramfs after interrupted kernel update.

### Solution
```bash
# Boot with older kernel from GRUB menu

# Rebuild initramfs for problematic kernel
dracut -f /boot/initramfs-$(uname -r).img $(uname -r)

# Or for specific kernel version
KERNEL_VERSION="3.10.0-1160.el7.x86_64"
dracut -f /boot/initramfs-${KERNEL_VERSION}.img ${KERNEL_VERSION}

# Verify GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg

# Check boot order
efibootmgr -v  # For UEFI systems

# Reboot
reboot
```

### Prevention
- Never interrupt kernel updates
- Maintain multiple kernel versions
- Regular backup of /boot partition
- Test kernel updates in non-production first

---

## Scenario 8: High Load Average but Low CPU Usage

### Problem
Load average shows 15+ but CPU usage is only 20%, system responsive.

### Diagnosis Steps
```bash
# Check load average
uptime
cat /proc/loadavg

# Check CPU usage
top
mpstat 1 5

# Check I/O wait
iostat -x 1 5

# Check disk performance
iotop

# Check processes in uninterruptible sleep
ps aux | awk '$8 ~ /D/'

# Check specific disk stats
vmstat 1 5
```

### Root Cause
Disk I/O bottleneck caused by failing hard drive, processes waiting for I/O.

### Solution
```bash
# Identify slow disk
hdparm -tT /dev/sda
smartctl -a /dev/sda

# Check for disk errors
dmesg | grep -i error
grep -i error /var/log/messages

# If RAID, check array status
cat /proc/mdstat

# Check for filesystem issues
df -h
mount | grep -i read-only

# Immediate mitigation: Identify and kill I/O-intensive processes
iotop -o -b -n 3

# Long-term: Replace failing disk
# Schedule maintenance window
# Use RAID rebuild or migrate to new disk
```

### Prevention
- Monitor disk health with SMART
- Set up I/O performance baselines
- Use RAID for redundancy
- Regular disk health checks

---

## Scenario 9: File System Corruption

### Problem
File system mounted as read-only, unable to write data.

### Diagnosis Steps
```bash
# Check mount status
mount | grep "ro,"
cat /proc/mounts

# Check filesystem errors
dmesg | grep -i "ext4\|xfs\|error"

# Check disk status
smartctl -a /dev/sda

# View filesystem details
tune2fs -l /dev/sda1  # For ext4
xfs_info /dev/sda1    # For XFS
```

### Root Cause
Unclean shutdown causing filesystem inconsistencies, disk errors.

### Solution
```bash
# Boot into single-user or rescue mode
# Add 'single' or '1' to kernel parameters at GRUB

# Unmount the filesystem
umount /dev/sda1

# Check and repair filesystem
# For ext4:
fsck.ext4 -y /dev/sda1
e2fsck -f -y /dev/sda1

# For XFS:
xfs_repair /dev/sda1

# If filesystem is still mounted (root):
# Remount as read-write
mount -o remount,rw /

# Reboot
reboot
```

### Prevention
- Implement UPS for clean shutdowns
- Regular filesystem checks via cron
- Monitor disk health
- Use journaling filesystems (ext4, XFS)

---

## Scenario 10: SELinux Blocking Application

### Problem
Application cannot access files/ports despite correct permissions, no errors in app logs.

### Diagnosis Steps
```bash
# Check SELinux status
getenforce
sestatus

# Check for denials
ausearch -m avc -ts recent
grep "denied" /var/log/audit/audit.log

# Check context of files
ls -Z /path/to/files

# Check process context
ps auxZ | grep application

# Check port context
semanage port -l | grep http
```

### Root Cause
Incorrect SELinux context on application files and custom port not labeled.

### Solution
```bash
# Fix file contexts
restorecon -Rv /var/www/html/

# Or set specific context
semanage fcontext -a -t httpd_sys_content_t "/custom/path(/.*)?"
restorecon -Rv /custom/path

# Allow custom port
semanage port -a -t http_port_t -p tcp 8080

# Generate and apply custom policy if needed
ausearch -m avc -ts recent | audit2allow -M myapp
semodule -i myapp.pp

# Check boolean settings
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
```

### Prevention
- Learn SELinux contexts for applications
- Use `audit2allow` to create policies
- Don't disable SELinux in production
- Test with permissive mode first

---

## Scenario 11: NFS Mount Issues

### Problem
NFS mount becomes unresponsive, applications hang when accessing mounted directory.

### Diagnosis Steps
```bash
# Check mount status
mount | grep nfs
df -h

# Check NFS service
systemctl status nfs-client.target

# Test NFS server connectivity
showmount -e nfs-server
rpcinfo -p nfs-server

# Check mount options
cat /etc/fstab | grep nfs
cat /proc/mounts | grep nfs

# Check for hung processes
ps aux | grep "D"
lsof +L1 | grep nfs
```

### Root Cause
NFS server became unavailable, hard mount causing processes to hang indefinitely.

### Solution
```bash
# Force unmount (if possible)
umount -f /mnt/nfs
umount -l /mnt/nfs  # Lazy unmount

# If unmount fails, kill processes
lsof /mnt/nfs
fuser -km /mnt/nfs

# Change to soft mount with timeout
vi /etc/fstab
```

Recommended NFS mount options:
```
nfs-server:/export /mnt/nfs nfs soft,timeo=10,retrans=2,intr 0 0
```

```bash
# Remount
mount -a

# Or temporary mount
mount -t nfs -o soft,timeo=10,retrans=2 nfs-server:/export /mnt/nfs
```

### Prevention
- Use soft mounts with timeouts for non-critical data
- Implement NFS server redundancy
- Monitor NFS server availability
- Use automount for on-demand mounting

---

## Scenario 12: Zombie Processes Accumulation

### Problem
Large number of zombie processes accumulating, ps shows many <defunct> processes.

### Diagnosis Steps
```bash
# Check for zombie processes
ps aux | grep defunct
ps aux | awk '$8 == "Z"'

# Count zombies
ps aux | grep -c defunct

# Find parent process
ps -eo pid,ppid,stat,cmd | grep Z

# Check parent process details
ps -p <PPID> -f
lsof -p <PPID>
```

### Root Cause
Parent process not properly reaping child processes, bug in application code.

### Solution
```bash
# Identify parent process
PPID=$(ps aux | grep defunct | awk '{print $3}' | head -1)

# Restart parent process
systemctl restart parent-service

# Or kill parent process (zombies will be cleaned by init)
kill -15 $PPID

# If parent is critical and can't be restarted:
# Zombies will be cleaned after parent exits
# Fix application code to properly handle SIGCHLD

# Check application code for proper wait() calls
```

### Prevention
- Properly implement signal handlers in applications
- Use `wait()` or `waitpid()` to reap children
- Monitor zombie process counts
- Regular code reviews for fork/exec patterns

---

## Scenario 13: DNS Resolution Failure

### Problem
System cannot resolve domain names, applications failing with DNS errors.

### Diagnosis Steps
```bash
# Test DNS resolution
nslookup google.com
dig google.com
host google.com

# Check DNS configuration
cat /etc/resolv.conf
cat /etc/nsswitch.conf

# Check if DNS servers are reachable
ping 8.8.8.8
nc -zvu 8.8.8.8 53

# Check local resolver
systemctl status systemd-resolved

# Test with different DNS server
nslookup google.com 8.8.8.8
```

### Root Cause
NetworkManager overwriting /etc/resolv.conf, corporate DNS servers unreachable.

### Solution
```bash
# Make resolv.conf immutable (temporary)
chattr +i /etc/resolv.conf

# Or configure NetworkManager properly
vi /etc/NetworkManager/NetworkManager.conf
```

Add:
```
[main]
dns=none
```

```bash
# Edit resolv.conf
vi /etc/resolv.conf
```

Add reliable DNS servers:
```
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1
```

```bash
# Restart NetworkManager
systemctl restart NetworkManager

# Or use nmcli
nmcli con mod "System eth0" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli con up "System eth0"

# Clear DNS cache
systemd-resolve --flush-caches
```

### Prevention
- Use reliable DNS servers (multiple)
- Monitor DNS resolution performance
- Implement local DNS caching (dnsmasq)
- Document DNS configuration in runbooks

---

## Scenario 14: Time Synchronization Problems

### Problem
System time drifting significantly, authentication failures due to time skew.

### Diagnosis Steps
```bash
# Check current time and timezone
date
timedatectl

# Check NTP status
timedatectl status
systemctl status chronyd

# Check NTP synchronization
chronyc tracking
chronyc sources -v

# Check time difference
ntpdate -q ntp-server
```

### Root Cause
Chronyd service stopped, system clock drifting, causing Kerberos authentication failures.

### Solution
```bash
# Start and enable chronyd
systemctl start chronyd
systemctl enable chronyd

# Configure NTP servers
vi /etc/chrony.conf
```

Add reliable NTP servers:
```
server 0.rhel.pool.ntp.org iburst
server 1.rhel.pool.ntp.org iburst
server 2.rhel.pool.ntp.org iburst
server 3.rhel.pool.ntp.org iburst
```

```bash
# Restart chronyd
systemctl restart chronyd

# Force immediate sync (if drift is large)
chronyd -q 'server 0.rhel.pool.ntp.org iburst'

# Or manually set time
timedatectl set-time "2024-01-15 14:30:00"
timedatectl set-timezone America/New_York

# Verify synchronization
chronyc tracking
```

### Prevention
- Always enable time synchronization
- Monitor time drift
- Use multiple NTP sources
- Set timezone correctly during installation

---

## Scenario 15: RPM Database Corruption

### Problem
YUM/DNF commands failing, unable to install or remove packages.

### Diagnosis Steps
```bash
# Check for corruption
rpm -qa
yum check

# Verify RPM database
rpm -vv --rebuilddb

# Check database files
ls -lh /var/lib/rpm/

# Look for lock files
ls -l /var/lib/rpm/__db*
```

### Root Cause
System crash during package installation, corrupted Berkeley DB files.

### Solution
```bash
# Backup RPM database
mkdir /root/rpm-backup
cp -r /var/lib/rpm /root/rpm-backup/

# Remove lock files
rm -f /var/lib/rpm/__db*

# Rebuild RPM database
rpm --rebuilddb

# If still corrupted, recover from backup
cd /var/lib
mv rpm rpm.corrupted
cp -r /root/rpm-backup/rpm .

# Verify
rpm -qa | wc -l
yum clean all
yum makecache
```

### Prevention
- Never kill yum/dnf processes forcefully
- Regular database verification
- Keep good backups of /var/lib/rpm
- Use transactions carefully

---

## Scenario 16: Firewall Blocking Legitimate Traffic

### Problem
Application cannot receive connections despite service running correctly.

### Diagnosis Steps
```bash
# Check firewall status
systemctl status firewalld
firewall-cmd --state

# List current rules
firewall-cmd --list-all
iptables -L -n -v

# Check specific port
firewall-cmd --query-port=8080/tcp
netstat -tulpn | grep 8080

# Test from client
telnet server_ip 8080
nc -zv server_ip 8080

# Check for connection tracking
conntrack -L
```

### Root Cause
Custom application port not added to firewall rules after service configuration change.

### Solution
```bash
# Add port to firewall (runtime)
firewall-cmd --add-port=8080/tcp

# Make permanent
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# Or add service
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# For specific source
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port port="8080" protocol="tcp" accept'
firewall-cmd --reload

# Verify
firewall-cmd --list-all
```

### Prevention
- Document all port requirements
- Use firewall zones appropriately
- Test firewall rules after changes
- Implement change management process

---

## Scenario 17: LVM Volume Group Issues

### Problem
Cannot extend logical volume, volume group showing as inactive.

### Diagnosis Steps
```bash
# Check LVM configuration
pvs
vgs
lvs

# Check volume group status
vgdisplay -v vg_name

# Scan for physical volumes
pvscan
vgscan
lvscan

# Check for missing devices
vgs -o +devices
```

### Root Cause
Physical volume became unavailable, volume group metadata inconsistent after disk failure.

### Solution
```bash
# Activate volume group
vgchange -ay vg_name

# If PV is missing, remove it from VG (DATA LOSS)
vgreduce --removemissing vg_name

# Restore from backup if available
vgcfgrestore -l vg_name
vgcfgrestore -f /etc/lvm/archive/vg_name_XXXXX.vg vg_name

# Extend LV (if space available)
lvextend -L +10G /dev/vg_name/lv_name
resize2fs /dev/vg_name/lv_name  # For ext4
xfs_growfs /mount/point          # For XFS

# Or extend to 100% free
lvextend -l +100%FREE /dev/vg_name/lv_name
```

### Prevention
- Use RAID for physical volumes
- Regular LVM configuration backups
- Monitor PV health
- Document LVM layout

---

## Scenario 18: Performance Degradation After Patching

### Problem
System performance significantly degraded after kernel update, applications slower.

### Diagnosis Steps
```bash
# Check current kernel
uname -r

# List installed kernels
rpm -qa | grep kernel

# Compare system performance
# Before and after metrics
vmstat 1 10
sar -A

# Check for new kernel parameters
diff /boot/grub2/grub.cfg.backup /boot/grub2/grub.cfg

# Review recent changes
rpm -qa --last | head -20
```

### Root Cause
New kernel version has performance regression, enabled mitigations for spectre/meltdown causing overhead.

### Solution
```bash
# Boot into previous kernel
# Edit GRUB at boot, select older kernel

# Set default kernel to older version
grub2-editenv list
grub2-set-default 1  # Index of previous kernel

# Or set specific kernel
grubby --set-default /boot/vmlinuz-3.10.0-1160.el7.x86_64

# Disable spectre/meltdown mitigations (assess risk first)
grubby --update-kernel=ALL --args="noibrs noibpb nopti nospectre_v2 nospectre_v1 l1tf=off nospec_store_bypass_disable no_stf_barrier mds=off tsx=on tsx_async_abort=off mitigations=off"

# Rebuild GRUB
grub2-mkconfig -o /boot/grub2/grub.cfg

# Reboot
reboot
```

### Prevention
- Test kernel updates in non-production
- Keep previous kernels installed
- Monitor performance baselines
- Have rollback plan ready

---

## Scenario 19: User Unable to Login

### Problem
User cannot login via SSH or console, receives "Permission denied" or account locked messages.

### Diagnosis Steps
```bash
# Check user account status
passwd -S username
chage -l username

# Check for account lock
faillock --user username

# Verify user exists
id username
getent passwd username

# Check SSH configuration
grep username /etc/ssh/sshd_config
cat /etc/ssh/sshd_config | grep -i allowusers

# Check PAM configuration
cat /etc/pam.d/sshd
cat /etc/pam.d/system-auth

# Review authentication logs
tail -50 /var/log/secure
journalctl -u sshd -n 50
```

### Root Cause
Account locked due to multiple failed login attempts, password expired.

### Solution
```bash
# Unlock account
faillock --user username --reset
pam_tally2 --user=username --reset  # For older systems

# Reset password expiry
chage -E -1 username
chage -M 99999 username

# Unlock account if locked
usermod -U username
passwd -u username

# Reset password
passwd username

# Check and fix home directory permissions
ls -ld /home/username
chmod 700 /home/username
chown username:username /home/username

# Verify SSH keys
ls -la /home/username/.ssh/
chmod 700 /home/username/.ssh
chmod 600 /home/username/.ssh/authorized_keys
```

### Prevention
- Configure reasonable password policies
- Monitor failed login attempts
- Implement account recovery procedures
- Use SSH keys instead of passwords
- Document user management procedures

---

## Scenario 20: Journal/Log Files Consuming Excessive Space

### Problem
/var/log partition full, systemd journal consuming 10+ GB of space.

### Diagnosis Steps
```bash
# Check journal size
journalctl --disk-usage

# Check log directory
du -sh /var/log/*
du -sh /var/log/journal/*/

# Find largest log files
find /var/log -type f -size +100M -exec ls -lh {} \;

# Check partition usage
df -h /var/log
```

### Root Cause
Journal retention not configured, excessive verbose logging by application.

### Solution
```bash
# Clean old journal entries
journalctl --vacuum-time=7d
journalctl --vacuum-size=500M

# Configure journal retention
vi /etc/systemd/journald.conf
```

Set appropriate limits:
```
[Journal]
SystemMaxUse=500M
SystemMaxFileSize=50M
MaxRetentionSec=7day
MaxFileSec=1day
```

```bash
# Restart journald
systemctl restart systemd-journald

# Rotate logs immediately
logrotate -f /etc/logrotate.conf

# Configure application logging
vi /etc/rsyslog.conf
# Reduce verbosity or redirect to separate partition
```

### Prevention
- Set journal size limits
- Configure log rotation properly
- Use separate /var/log partition
- Monitor log growth
- Archive old logs to external storage

---

## Additional Resources

### Useful Commands Reference

```bash
# System monitoring
top, htop, atop
sar -A
vmstat, iostat, mpstat
nmon

# Log analysis
journalctl -xe
tail -f /var/log/messages
grep -r "error" /var/log/

# Network troubleshooting
ss -tulpn
netstat -tulpn
tcpdump -i eth0
mtr, traceroute

# Disk management
lsblk, blkid
smartctl -a /dev/sda
hdparm -tT /dev/sda
```

### Best Practices

1. **Always backup before making changes**
2. **Test in non-production first**
3. **Document everything**
4. **Use version control for configs**
5. **Implement monitoring and alerting**
6. **Maintain runbooks for common issues**
7. **Regular system audits**
8. **Keep systems patched**
9. **Use configuration management tools**
10. **Practice disaster recovery**

---


## Author

**Sajal Jana**  
Linux System Administrator  
Specializing in RHEL production environments

---

*Last Updated: February 2026*
