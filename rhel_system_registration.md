RHEL Cheat Sheet: subscription-manager & dnf
--------------------------------------------

SUBSCRIPTION-MANAGER (Register & Repo Control)
```
# Register system
subscription-manager register

# Attach subscription automatically
subscription-manager attach --auto

# List available subscriptions
subscription-manager list --available

# List consumed subscriptions
subscription-manager list --consumed

# Show current status
subscription-manager status

# Show identity (system info)
subscription-manager identity

# List enabled repositories
subscription-manager repos --list-enabled

# Enable a repository
subscription-manager repos --enable=<repo_id>

# Disable a repository
subscription-manager repos --disable=<repo_id>

# Unregister system
subscription-manager unregister
```

DNF (Package Management)
```
# Install package
dnf install <package>

# Remove package
dnf remove <package>

# Update all packages
dnf update

# Update specific package
dnf update <package>

# Check for updates
dnf check-update

# Search package
dnf search <keyword>

# Get package info
dnf info <package>

# List installed packages
dnf list installed

# List available packages
dnf list available

# Show enabled repos
dnf repolist

# Clean cache
dnf clean all

# Rebuild cache
dnf makecache

# View history
dnf history

# Undo last transaction
dnf history undo last
```

NOTES:
- subscription-manager = registration & repo access
- dnf = actual package installation/management
- Both are tightly linked in RHEL systems
