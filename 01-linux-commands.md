# Linux Commands Cheat Sheet

> Essential Linux commands for DevOps engineers — files, processes, networking, systemd, packages, users, disks, and shell utilities.

---

## Table of Contents

- [File & Directory Management](#1-file--directory-management)
- [Viewing & Processing Files](#2-viewing--processing-files)
- [Permissions & Ownership](#3-permissions--ownership)
- [Archives & Compression](#4-archives--compression)
- [Process Management](#5-process-management)
- [System Information](#6-system-information)
- [Networking](#7-networking)
- [systemd & Services](#8-systemd--services)
- [Package Management](#9-package-management)
- [Users & Groups](#10-users--groups)
- [Disk & Storage](#11-disk--storage)
- [Shell & Environment](#12-shell--environment)
- [Cron & Scheduling](#13-cron--scheduling)
- [Useful One-Liners](#14-useful-one-liners)

---

## 1. File & Directory Management

```
pwd                        # Print current directory
ls -lah                    # Long list, all files, human-readable sizes
cd /path/to/dir            # Change directory
mkdir -p dir/sub/dir       # Create nested directories
touch file.txt             # Create empty file / update timestamp
cp -r source/ dest/        # Copy recursively
mv old.txt new.txt         # Move / rename
rm -rf dir/                # Remove recursively (CAREFUL!)
find /var -name "*.log"    # Find by name
find /var -name "*.log" -mtime +7 -delete   # Older than 7 days, delete
find . -type f -size +100M  # Files larger than 100 MB
find / -perm -4000 2>/dev/null              # Find SUID binaries
du -sh */                  # Disk usage per subdirectory
df -h                      # Filesystem usage (human-readable)
file script.sh             # Identify file type
stat file.txt              # Detailed file metadata
tree -L 2                  # Directory tree, depth 2
```

## 2. Viewing & Processing Files

```
cat file.txt               # Print whole file
less +F app.log            # Paginate / live-follow
head -n 50 file.txt        # First 50 lines
tail -n 100 -f app.log     # Last 100 lines, follow
wc -l file.txt             # Count lines
sort -u users.txt          # Sort, unique
uniq -c                    # Count adjacent duplicates
diff -u old.conf new.conf  # Compare files
grep -rni "pattern" ./dir  # Recursive, line numbers, case-insensitive
grep -c "ERROR" app.log    # Count matches
sed -i 's/old/new/g' file.txt              # In-place replace
sed -n '10,20p' file.txt                   # Print lines 10-20
awk '{print $1, $3}' access.log            # Print columns 1 and 3
awk -F: '{print $1}' /etc/passwd           # Use custom delimiter
cut -d',' -f1,3 data.csv                   # Cut CSV columns
tr 'a-z' 'A-Z' < file.txt                  # Translate characters
cat file.txt | xargs -n1 echo              # Build commands from stdin
```

## 3. Permissions & Ownership

```
chmod 755 script.sh        # rwx r-x r-x (owner full, others read/exec)
chmod u+x,g-w,o-r file     # Symbolic: add exec user, remove write group, remove read others
chmod -R 644 /var/www      # Recursive
chown user:group file.txt  # Change owner and group
chown -R www-data:www-data /var/www
chgrp developers file.txt  # Change group only
umask 022                  # Default permission mask
id                         # Current user identity
whoami                     # Current user
sudo -i                    # Root shell
su - username              # Switch user with login shell
```

**Numeric permission bits:** `r=4, w=2, x=1` → owner/group/others (e.g., `755` = owner `rwx`, group `r-x`, others `r-x`).

## 4. Archives & Compression

```
tar -czvf archive.tar.gz dir/     # Create gzip archive
tar -xzvf archive.tar.gz          # Extract
tar -xzvf archive.tar.gz -C /opt  # Extract to specific directory
tar -tvf archive.tar.gz           # List contents
tar -cjvf archive.tar.bz2 dir/    # bzip2 archive
gzip file.txt                     # Compress to .gz
gunzip file.txt.gz                # Decompress
zip -r archive.zip dir/           # ZIP archive
unzip archive.zip -d /opt         # Extract ZIP
```

## 5. Process Management

```
ps aux                     # All processes
ps aux | grep nginx        # Filter
ps -ef --forest            # Tree view
top                        # Live process viewer
htop                       # Interactive process viewer
kill PID                   # Send SIGTERM
kill -9 PID                # Force kill (SIGKILL)
kill -HUP PID              # Reload signal
pkill -f "python app.py"   # Kill by name pattern
killall nginx              # Kill by process name
jobs                       # Background jobs
bg %1                      # Resume job in background
fg %1                      # Bring job to foreground
nohup ./app.sh &           # Immune to hangup
nice -n 10 command         # Lower priority
renice -n 5 -p PID         # Change priority of running process
watch -n2 'ps aux | head'  # Repeat command every 2s
```

## 6. System Information

```
uname -a                   # Kernel info
uptime                     # Load average, uptime
free -h                    # Memory usage
lscpu                      # CPU details
lsblk                      # Block devices
lspci / lsusb              # PCI / USB devices
dmidecode                  # Hardware DMI info (root)
cat /etc/os-release        # Distribution info
date                       # Current date/time
timedatectl                # Time & timezone status
hostnamectl                # Hostname & system info
```

## 7. Networking

```
ip addr                    # Interface addresses
ip route                   # Routing table
ip link set eth0 up        # Bring interface up
ss -tulpn                  # Listening ports + processes
ping -c4 google.com        # Connectivity test
traceroute example.com     # Path trace
dig +short example.com     # DNS lookup
nslookup example.com       # DNS lookup
curl -I https://example.com            # HTTP headers
curl -o file.zip -L https://...        # Download
curl -X POST -H "Content-Type: application/json" -d '{}' URL
wget -q https://...                    # Download
ssh -p 2222 -i key.pem user@host       # SSH with port & key
scp -r ./local user@host:/remote/path  # Copy over SSH
rsync -avz --progress ./dir user@host:/backup/   # Sync over SSH
nc -zv host 443            # Port reachability test
ethtool eth0               # Interface statistics
hostname -I                # Show IP addresses
```

## 8. systemd & Services

```
systemctl start nginx              # Start service
systemctl stop nginx               # Stop
systemctl restart nginx            # Restart
systemctl reload nginx             # Reload config (no downtime)
systemctl enable nginx             # Start on boot
systemctl disable nginx            # Remove from boot
systemctl status nginx             # Status + recent logs
systemctl is-active nginx          # Check running
systemctl is-enabled nginx         # Check boot-enabled
systemctl daemon-reload            # Reload unit files after changes
systemctl list-units --type=service --state=running
systemctl edit nginx               # Drop-in override editor
systemctl cat nginx                # Show unit file

journalctl -u nginx -f             # Follow service logs
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err         # Errors only
journalctl -b                      # Logs since boot
journalctl -b -1                   # Logs from previous boot
```

## 9. Package Management

**Debian/Ubuntu (apt):**
```
apt update                 # Refresh package index
apt upgrade                # Upgrade all packages
apt install nginx          # Install
apt install -y nginx       # Auto-confirm
apt remove nginx           # Remove (keep config)
apt purge nginx            # Remove + purge config
apt autoremove             # Clean orphaned packages
apt search nginx           # Search
apt show nginx             # Package details
dpkg -l | grep nginx       # List installed
dpkg -i package.deb        # Install local .deb
```

**RHEL/CentOS/Fedora (dnf/yum):**
```
dnf check-update           # Refresh + list updates
dnf update                 # Upgrade all
dnf install nginx -y       # Install
dnf remove nginx           # Remove
dnf search nginx           # Search
dnf info nginx             # Details
rpm -qa | grep nginx       # List installed
rpm -ivh package.rpm       # Install local .rpm
```

## 10. Users & Groups

```
useradd -m -s /bin/bash john     # Create user with home & shell
usermod -aG sudo john            # Add user to group
userdel -r john                  # Delete user + home
groupadd developers              # Create group
groups john                      # Show user groups
passwd john                      # Set password
chage -l john                    # Password aging info
who                              # Logged-in users
last                             # Login history
su - john                        # Switch to user
visudo                           # Edit sudoers safely
```

## 11. Disk & Storage

```
df -h                      # Filesystem usage
du -sh /var/*              # Per-directory usage
lsblk -f                   # Devices + filesystems
blkid                      # UUIDs
mount /dev/sdb1 /mnt/data  # Mount
umount /mnt/data           # Unmount
fdisk -l                   # Partition tables
mkfs.ext4 /dev/sdb1        # Format partition
mount -a                   # Mount everything in /etc/fstab
```

## 12. Shell & Environment

```
export VAR=value           # Set env variable
env                        # Show all variables
alias ll='ls -lah'         # Create alias (add to ~/.bashrc)
history | grep docker      # Search history
!128                       # Re-run command 128
which docker               # Path of command
whereis docker             # Binary, source, man pages
man ls                     # Manual page
echo $PATH
source ~/.bashrc           # Reload shell config
```

## 13. Cron & Scheduling

```
crontab -e                 # Edit current user's crontab
crontab -l                 # List crontab
```

```
# ┌─ minute (0-59)
# │ ┌─ hour (0-23)
# │ │ ┌─ day of month (1-31)
# │ │ │ ┌─ month (1-12)
# │ │ │ │ ┌─ day of week (0-7, Sun=0,7)
# │ │ │ │ │
  0 2 * * *   /opt/scripts/backup.sh      # Daily at 2:00 AM
*/15 * * * *  /opt/scripts/monitor.sh     # Every 15 minutes
  0 0 * * 0   /opt/scripts/weekly.sh      # Weekly on Sunday
```

## 14. Useful One-Liners

```bash
# Largest files in current directory
du -ah . | sort -rh | head -20

# Total size of a directory
du -sh /var/log

# Count open file descriptors for a process
ls /proc/PID/fd | wc -l

# Watch disk space in real time
watch -n5 'df -h'

# Find and kill process on port 8080
kill $(lsof -t -i :8080)

# Show established connections per service
ss -s

# Quick HTTP server (Python)
python3 -m http.server 8000

# Show all listening ports with process names
ss -tulpn | grep LISTEN
```

---
