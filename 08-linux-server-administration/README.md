# Project 08 — Linux Server Administration

## What You Will Learn
- Essential Linux commands for server administration
- How to use the Azure Portal's Serial Console and Run Command for Linux VMs
- User, group, and file permission management
- Service management with systemd
- Package management with apt
- Process monitoring, disk management, and networking
- Log analysis and bash scripting for automation

---

## Background — Read This First

Linux is the dominant OS for servers, containers, and cloud workloads. The job spec explicitly lists "Windows and Linux systems" — you will manage both. Everything in Linux is a file. Most configuration lives in text files under `/etc/`. You manage almost everything from the command line.

**Key principle:** Master the terminal. Linux admins who rely on GUIs are limited. The SSH terminal is how you manage Linux — locally, remotely, and in Azure.

---

## Prerequisites
- Azure subscription, CLI logged in
- SSH key pair created (Project 02, Step 1)

---

## Step 1 — Deploy an Ubuntu Server VM

### Portal (GUI)
Create a VM (Project 02 steps) with:
- Image: `Ubuntu Server 22.04 LTS`
- Size: `Standard_B2s`
- Authentication: SSH public key (paste your `~/.ssh/azure_lab_key.pub`)
- Resource group: `rg-linux-lab`
- Allow SSH (port 22)

### CLI
```bash
az group create --name rg-linux-lab --location uksouth

az vm create \
  --resource-group rg-linux-lab \
  --name vm-linux-lab \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --ssh-key-value ~/.ssh/azure_lab_key.pub \
  --public-ip-sku Standard

LINUX_IP=$(az vm show \
  --resource-group rg-linux-lab \
  --name vm-linux-lab \
  --show-details --query publicIps --output tsv)
echo "Linux VM IP: $LINUX_IP"
```

---

## Step 2 — Connect to the VM

### Portal (GUI) — Azure Serial Console (emergency access, no network needed)
1. Navigate to `vm-linux-lab`
2. In the left panel, click **Serial console** (under Help)
3. Wait for the console to initialise — you get a login prompt
4. This works even if SSH is locked out or the VM has no public IP

### Portal (GUI) — Run Command
1. Navigate to `vm-linux-lab` → **Run command** → **RunShellScript**
2. Type a command and click **Run** — output appears below

### CLI / SSH
```bash
ssh -i ~/.ssh/azure_lab_key azureuser@$LINUX_IP
```

All commands from Step 3 onward are run **inside the VM** (via SSH, Serial Console, or Run Command).

---

## Step 3 — Essential File System Navigation

```bash
pwd                          # Where am I?
ls -lh                       # List files (long format, human-readable sizes)
ls -lah                      # List all files including hidden
cd /etc && ls | head -20     # Go to /etc (where config files live)
cd ~                         # Go back to home directory

# Find a file
find /etc -name "*.conf" 2>/dev/null | head -10

# Search text inside files
grep -r "sshd" /etc/ 2>/dev/null | head -5

# Read a file
cat /etc/hostname            # Print file to screen
less /etc/passwd             # Read page by page (q to quit)
head -5 /var/log/syslog      # First 5 lines
tail -20 /var/log/syslog     # Last 20 lines
tail -f /var/log/syslog      # Follow live (Ctrl+C to stop)
```

**What you just learned:** `tail -f` is one of the most-used commands in daily ops — you run it during an incident to watch logs in real time. `less` is how you read long files without printing everything to screen.

---

## Step 4 — User and Group Management

```bash
whoami                       # Who am I?
id                           # My user ID, group IDs
cat /etc/passwd | cut -d: -f1  # List all users

# Create a new user with home directory and bash shell
sudo useradd -m -s /bin/bash jsmith
sudo passwd jsmith           # Set a password

# Verify the user
id jsmith
grep jsmith /etc/passwd

# Create a group
sudo groupadd infrastructure

# Add user to the group
sudo usermod -aG infrastructure jsmith

# Give user sudo access
sudo usermod -aG sudo jsmith

# Verify group membership
id jsmith
groups jsmith

# Switch to the user
su - jsmith
whoami
exit

# Lock (disable) an account
sudo usermod -L jsmith
# Unlock
sudo usermod -U jsmith

# Delete user and home directory
sudo userdel -r jsmith
```

**What you just learned:** `/etc/passwd` stores accounts. `/etc/shadow` stores password hashes (root-only). `/etc/group` stores groups. When someone leaves an organisation, `usermod -L` disables their account immediately — `userdel -r` removes it and their home directory.

---

## Step 5 — File Permissions

```bash
# Create test files
touch testfile.txt
mkdir testdir

# View permissions
ls -l testfile.txt
# -rw-rw-r-- 1 azureuser azureuser 0 Jun 23 10:00 testfile.txt
#  ^  ^  ^
#  |  |  └── others: r-- (read only)
#  |  └───── group: rw- (read/write)
#  └──────── owner: rw- (read/write)

# Change permissions (octal notation)
chmod 644 testfile.txt    # owner: rw-, group: r--, others: r--
chmod 755 testdir         # owner: rwx, group: r-x, others: r-x
chmod 600 testfile.txt    # owner: rw-, nobody else (private file, e.g. SSH key)

# Change permissions (symbolic notation)
chmod u+x testfile.txt    # add execute for owner
chmod g-w testfile.txt    # remove write for group
chmod o+r testfile.txt    # add read for others

# Change owner
sudo chown jsmith testfile.txt
sudo chown jsmith:infrastructure testfile.txt   # user:group

# Recursive ownership change
sudo chown -R jsmith:infrastructure testdir/
```

**Permission reference:**
| Number | Meaning |
|---|---|
| 7 | rwx — read, write, execute |
| 6 | rw- — read, write |
| 5 | r-x — read, execute |
| 4 | r-- — read only |
| 0 | --- — no permissions |

**What you just learned:** SSH private keys must be `600` (owner read/write only) — SSH will refuse to use a key with looser permissions. Config files are typically `644`. Scripts and executables are `755`. Getting permissions wrong causes both security vulnerabilities and broken functionality.

---

## Step 6 — Package Management

```bash
# Update the package index (always do this first)
sudo apt update

# See what needs updating
apt list --upgradeable 2>/dev/null | head -10

# Install a package
sudo apt install -y nginx

nginx -v   # Verify

# Remove a package
sudo apt remove nginx -y

# Remove package AND its config files
sudo apt purge nginx -y

# Remove unused dependencies
sudo apt autoremove -y

# Search for a package
apt search "web server" 2>/dev/null | head -10

# Show package information
apt show nginx 2>/dev/null
```

**What you just learned:** Always run `apt update` before installing — it refreshes the list of available versions. `apt upgrade` updates all installed packages. In production, always test upgrades in a staging environment first, especially for critical packages like the Linux kernel.

---

## Step 7 — Service Management with systemd

```bash
# Install nginx for practice
sudo apt install -y nginx

# Check service status
sudo systemctl status nginx

# Start / Stop / Restart / Reload
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx      # stops then starts (brief downtime)
sudo systemctl reload nginx       # re-reads config, no downtime (nginx supports this)

# Enable / Disable (controls start on boot)
sudo systemctl enable nginx       # starts automatically on boot
sudo systemctl disable nginx      # does NOT start on boot
sudo systemctl is-enabled nginx   # check enabled state

# List failed services
sudo systemctl list-units --type=service --state=failed

# View service logs
sudo journalctl -u nginx --no-pager | tail -20
sudo journalctl -u nginx -f        # follow live (Ctrl+C to stop)
sudo journalctl -p err --since "1 hour ago"  # errors in last hour
```

**What you just learned:** `restart` stops and starts the service — there is a brief interruption. `reload` makes the process re-read its config without stopping — zero downtime. Always use `reload` in production for services that support it (nginx, Apache, sshd).

---

## Step 8 — Process Monitoring

```bash
# Real-time process viewer
top
# (press q to quit, Shift+M sort by memory, Shift+P sort by CPU)

# Better: install htop
sudo apt install -y htop
htop
# (press q to quit)

# List all processes
ps aux

# Find a specific process
ps aux | grep nginx
pgrep nginx           # just the PID

# Kill a process
kill PID              # graceful (SIGTERM)
kill -9 PID           # force (SIGKILL)
pkill nginx           # kill by name

# Memory usage
free -h

# Disk usage
df -h
du -sh /var/log/      # size of a directory
du -sh /var/log/* | sort -rh | head -10  # biggest items in /var/log
```

---

## Step 9 — Log Management

```bash
# System messages
sudo tail -50 /var/log/syslog

# Authentication log (login attempts, sudo usage)
sudo tail -50 /var/log/auth.log

# Find SSH brute force attempts
sudo grep "Failed password" /var/log/auth.log | tail -20
sudo grep "Failed password" /var/log/auth.log | wc -l

# Find successful SSH logins
sudo grep "Accepted" /var/log/auth.log | tail -10

# Nginx access log
sudo tail -20 /var/log/nginx/access.log

# journalctl — systemd's unified log
sudo journalctl --since "1 hour ago" | tail -50
sudo journalctl --since "2026-06-23 09:00" --until "2026-06-23 10:00"
sudo journalctl -p 3 --since "24 hours ago"  # priority 3 = errors only
```

**What you just learned:** `auth.log` is your security log. Hundreds of failed SSH passwords per hour = brute force attack in progress. Check if any eventually succeeded (`Accepted`). If yes: treat as a security incident — change keys, check for new user accounts, check cron jobs.

---

## Step 10 — Networking from the Command Line

```bash
# Show network interfaces and IP addresses
ip addr show
ip addr show eth0

# Show routing table
ip route show

# Check listening ports
sudo ss -tulnp
# -t=TCP -u=UDP -l=listening -n=numeric -p=show process

# Test connectivity
curl -v http://localhost        # test local HTTP
ping google.com -c 4           # test internet
nc -zv google.com 443          # test if port is open

# DNS lookup
nslookup google.com
dig google.com A

# Trace network path
traceroute google.com

# Check UFW firewall
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

---

## Step 11 — Write a Health-Check Bash Script

```bash
cat > ~/health-check.sh << 'EOF'
#!/bin/bash
echo "==============================="
echo " SERVER HEALTH REPORT"
echo " $(date)"
echo " Hostname: $(hostname)"
echo "==============================="

echo ""
echo "--- OS ---"
grep PRETTY_NAME /etc/os-release | cut -d= -f2

echo ""
echo "--- Uptime ---"
uptime

echo ""
echo "--- CPU ---"
echo "Load average: $(cut -d' ' -f1-3 /proc/loadavg)"
echo "CPU cores: $(nproc)"

echo ""
echo "--- Memory ---"
free -h | awk 'NR==2{printf "Used: %s / Total: %s\n", $3, $2}'

echo ""
echo "--- Disk ---"
df -h | grep "^/dev" | awk '{printf "%-20s %5s used / %5s total (%s)\n", $6, $3, $2, $5}'

echo ""
echo "--- Top 5 CPU Processes ---"
ps aux --sort=-%cpu | awk 'NR>1 && NR<=6 {printf "%-25s %5s%%\n", $11, $3}'

echo ""
echo "--- Failed Services ---"
systemctl list-units --type=service --state=failed --no-pager 2>/dev/null || echo "None"

echo ""
echo "--- Recent Auth Failures ---"
FAIL_COUNT=$(sudo grep -c "Failed password" /var/log/auth.log 2>/dev/null || echo "0")
echo "SSH failed attempts: $FAIL_COUNT"

echo "==============================="
EOF

chmod +x ~/health-check.sh
bash ~/health-check.sh
```

**Schedule it with cron:**
```bash
crontab -e
# Add this line (runs at 7am weekdays):
# 0 7 * * 1-5 /home/azureuser/health-check.sh >> /home/azureuser/health-check.log 2>&1
```

**What you just learned:** This script is exactly what you would run each morning on production servers. In a mature team, it runs automatically and the output is mailed to the team or fed into a monitoring system.

---

## Step 12 — Exit and Clean Up

```bash
exit  # leave the SSH session
```

### Portal (GUI)
Navigate to `rg-linux-lab` → **Delete resource group** → confirm

### CLI
```bash
az group delete --name rg-linux-lab --yes --no-wait
```

---

## Key Commands Cheat Sheet

| Category | Command | What It Does |
|---|---|---|
| Navigation | `ls -lah`, `cd`, `find`, `grep -r` | List, navigate, search |
| Files | `cat`, `less`, `head`, `tail -f` | Read files, follow logs |
| Permissions | `chmod 644`, `chown user:group` | Control access |
| Users | `useradd -m`, `usermod -aG`, `passwd` | Manage accounts |
| Packages | `apt update && apt install` | Install software |
| Services | `systemctl start/stop/status/enable` | Manage daemons |
| Processes | `ps aux`, `top`, `kill`, `pgrep` | Monitor and control |
| Disk | `df -h`, `du -sh`, `lsblk` | Check storage |
| Network | `ip addr`, `ss -tulnp`, `curl`, `dig` | Check connectivity |
| Logs | `tail -f /var/log/syslog`, `journalctl -u` | Monitor logs |

---

## Common Interview Questions

1. **"A Linux server is running slowly. How do you diagnose it?"** — `top` or `htop` for CPU/memory. `df -h` for disk space. `sudo ss -tulnp` for unusual network connections. `sudo journalctl -p 3 --since "1 hour ago"` for errors. If disk I/O is suspected, `iostat -x 2` (needs `sysstat` package) shows per-disk utilisation.

2. **"How do you make a service start automatically after a reboot?"** — `sudo systemctl enable <service-name>`. This creates a symlink in the appropriate systemd target directory. Verify with `systemctl is-enabled <service>`.

3. **"What does chmod 755 mean?"** — Owner gets read/write/execute (7). Group gets read/execute (5). Others get read/execute (5). Used for scripts and directories. Compare to `644` (owner rw, everyone else r) used for config files and web content.

---

## Next Steps
Move on to **Project 09 — Kubernetes & AKS**.
