# 1. Foundational Skills: Linux & Networking

## Table of Contents
- [Linux Installation & Setup](#linux-installation--setup)
- [Essential Linux Commands](#essential-linux-commands)
- [File Permissions & Ownership](#file-permissions--ownership)
- [User Management](#user-management)
- [Process Management](#process-management)
- [Package Management](#package-management)
- [Shell Scripting](#shell-scripting)
- [Networking Basics](#networking-basics)
- [SSH Configuration](#ssh-configuration)
- [Firewall Setup](#firewall-setup)
- [Cron Jobs](#cron-jobs)
- [Projects](#projects)

---

## Linux Installation & Setup

### Option 1: VirtualBox VM (Recommended for Beginners)

```bash
# 1. Download VirtualBox from virtualbox.org
# 2. Download Ubuntu Server 22.04 LTS ISO from ubuntu.com
# 3. Create a new VM in VirtualBox:
#    - Name: ubuntu-devops
#    - Type: Linux
#    - Version: Ubuntu (64-bit)
#    - RAM: 2048 MB
#    - Disk: 20 GB (dynamically allocated)
# 4. Mount the ISO and install
```

### Option 2: Using Vagrant (Automated VM Setup)

```bash
# Install Vagrant
brew install vagrant    # macOS
sudo apt install vagrant  # Ubuntu

# Create a Vagrantfile
mkdir ~/devops-lab && cd ~/devops-lab

cat > Vagrantfile << 'EOF'
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.hostname = "devops-lab"
  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end
end
EOF

# Start the VM
vagrant up

# SSH into the VM
vagrant ssh
```

### Option 3: WSL2 (Windows Users)

```powershell
# Open PowerShell as Admin
wsl --install -d Ubuntu-22.04

# After restart, set username and password
# Launch Ubuntu from Start Menu
```

---

## Essential Linux Commands

### Navigation & File Operations

```bash
# Print current directory
pwd
# Output: /home/student

# List files (detailed view)
ls -la
# Output:
# drwxr-xr-x 2 student student 4096 Jan 15 10:00 .
# drwxr-xr-x 3 root    root    4096 Jan 15 09:00 ..
# -rw-r--r-- 1 student student  220 Jan 15 09:00 .bash_logout

# Change directory
cd /var/log          # Go to /var/log
cd ~                 # Go to home directory
cd ..                # Go up one level
cd -                 # Go to previous directory

# Create directories
mkdir projects                    # Single directory
mkdir -p projects/app/src         # Nested directories (creates parents)

# Create files
touch index.html                  # Create empty file
echo "Hello World" > hello.txt    # Create file with content
echo "More text" >> hello.txt     # Append to file

# Copy files and directories
cp file.txt backup.txt            # Copy file
cp -r projects/ projects_backup/  # Copy directory recursively

# Move / Rename
mv old_name.txt new_name.txt      # Rename
mv file.txt /tmp/                 # Move to /tmp

# Remove
rm file.txt                       # Remove file
rm -r directory/                  # Remove directory recursively
rm -rf directory/                 # Force remove (use with caution!)
```

### Viewing & Searching Files

```bash
# View file content
cat file.txt              # Print entire file
less file.txt             # Scrollable view (q to quit)
head -20 file.txt         # First 20 lines
tail -20 file.txt         # Last 20 lines
tail -f /var/log/syslog   # Follow log in real-time

# Search inside files
grep "error" logfile.txt              # Find lines with "error"
grep -i "error" logfile.txt           # Case-insensitive
grep -r "TODO" /home/student/projects # Search recursively in directory
grep -n "error" logfile.txt           # Show line numbers
grep -c "error" logfile.txt           # Count matches

# Find files
find / -name "nginx.conf"             # Find by name
find /home -type f -name "*.log"      # Find .log files
find /tmp -type f -mtime +7           # Files modified more than 7 days ago
find . -size +100M                    # Files larger than 100MB

# Text processing
cat access.log | awk '{print $1}'     # Print first column
cat file.txt | sed 's/old/new/g'      # Replace text
cat file.txt | sort                   # Sort lines
cat file.txt | sort | uniq            # Sort and remove duplicates
cat file.txt | wc -l                  # Count lines
```

### Disk & System Info

```bash
# Disk usage
df -h                    # Disk space (human-readable)
du -sh /var/log          # Size of a directory
du -sh * | sort -rh      # Size of each item, sorted

# System info
uname -a                 # Kernel info
lsb_release -a           # OS version
hostname                 # Machine name
uptime                   # How long system has been running
free -h                  # Memory usage
```

---

## File Permissions & Ownership

### Understanding Permission Notation

```
-rwxr-xr-- 1 student devops 4096 Jan 15 10:00 script.sh
│├──┤├──┤├──┤   │       │
│ │   │   │     │       └── Group
│ │   │   │     └────────── Owner
│ │   │   └──────────────── Others (r--)  = 4
│ │   └──────────────────── Group  (r-x)  = 5
│ └──────────────────────── Owner  (rwx)  = 7
└────────────────────────── File type (- = file, d = directory)

r = read (4)    w = write (2)    x = execute (1)
```

### Changing Permissions

```bash
# Numeric method
chmod 755 script.sh     # rwxr-xr-x (owner: all, group: read+exec, others: read+exec)
chmod 644 config.txt    # rw-r--r-- (owner: read+write, group: read, others: read)
chmod 600 secret.key    # rw------- (only owner can read+write)

# Symbolic method
chmod +x script.sh      # Add execute for all
chmod u+x script.sh     # Add execute for owner only
chmod g-w file.txt      # Remove write for group
chmod o-rwx file.txt    # Remove all permissions for others

# Recursive (apply to directory and all contents)
chmod -R 755 /var/www/html
```

### Changing Ownership

```bash
# Change owner
chown student file.txt

# Change owner and group
chown student:devops file.txt

# Recursive ownership change
chown -R www-data:www-data /var/www/html
```

### Example: Setting Up a Web Directory

```bash
# Create web directory
sudo mkdir -p /var/www/mysite

# Set proper ownership (web server user)
sudo chown -R www-data:www-data /var/www/mysite

# Set proper permissions
sudo chmod -R 755 /var/www/mysite

# Verify
ls -la /var/www/
# drwxr-xr-x 2 www-data www-data 4096 Jan 15 10:00 mysite
```

---

## User Management

```bash
# Add a new user
sudo useradd -m -s /bin/bash devuser
# -m = create home directory
# -s = set default shell

# Set password
sudo passwd devuser

# Add user to a group
sudo usermod -aG sudo devuser    # Add to sudo group
sudo usermod -aG docker devuser  # Add to docker group

# View user info
id devuser
# Output: uid=1001(devuser) gid=1001(devuser) groups=1001(devuser),27(sudo)

# List all users
cat /etc/passwd

# Delete a user
sudo userdel -r devuser    # -r removes home directory

# Switch user
su - devuser

# Run command as another user
sudo -u www-data whoami
```

---

## Process Management

```bash
# View running processes
ps aux                       # All processes (detailed)
ps aux | grep nginx          # Filter for nginx

# Real-time process monitor
top                          # Basic monitor
htop                         # Enhanced monitor (install: sudo apt install htop)

# Background & Foreground
./long_script.sh &           # Run in background
jobs                         # List background jobs
fg %1                        # Bring job 1 to foreground
bg %1                        # Send job 1 to background

# Kill processes
kill 1234                    # Graceful kill (SIGTERM)
kill -9 1234                 # Force kill (SIGKILL)
killall nginx                # Kill all nginx processes
pkill -f "python app.py"    # Kill by command pattern

# System services (systemd)
sudo systemctl start nginx       # Start service
sudo systemctl stop nginx        # Stop service
sudo systemctl restart nginx     # Restart service
sudo systemctl status nginx      # Check status
sudo systemctl enable nginx      # Start on boot
sudo systemctl disable nginx     # Don't start on boot

# View service logs
journalctl -u nginx              # All nginx logs
journalctl -u nginx -f           # Follow nginx logs
journalctl -u nginx --since "1 hour ago"
```

---

## Package Management

### Ubuntu/Debian (apt)

```bash
# Update package list
sudo apt update

# Upgrade installed packages
sudo apt upgrade -y

# Install a package
sudo apt install -y nginx curl wget git

# Remove a package
sudo apt remove nginx
sudo apt purge nginx         # Remove + delete config files
sudo apt autoremove          # Remove unused dependencies

# Search for packages
apt search nginx

# Show package info
apt show nginx
```

### CentOS/RHEL (yum/dnf)

```bash
# Update packages
sudo yum update -y       # CentOS 7
sudo dnf update -y       # CentOS 8+

# Install
sudo yum install -y nginx
sudo dnf install -y nginx

# Remove
sudo yum remove nginx

# Search
yum search nginx
```

---

## Shell Scripting

### Basics

```bash
#!/bin/bash
# Save as: hello.sh
# Run: chmod +x hello.sh && ./hello.sh

# Variables
NAME="DevOps Student"
COURSE="DevOps Roadmap"
echo "Hello, $NAME! Welcome to $COURSE."

# Read user input
echo "Enter your name:"
read USER_NAME
echo "Hello, $USER_NAME!"

# Command substitution
CURRENT_DATE=$(date +"%Y-%m-%d")
HOSTNAME=$(hostname)
echo "Date: $CURRENT_DATE, Host: $HOSTNAME"
```

### Conditionals

```bash
#!/bin/bash

# If-else
FILE="/etc/nginx/nginx.conf"
if [ -f "$FILE" ]; then
    echo "Nginx config exists."
else
    echo "Nginx is not installed."
fi

# Numeric comparison
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
if [ "$DISK_USAGE" -gt 80 ]; then
    echo "WARNING: Disk usage is ${DISK_USAGE}%!"
elif [ "$DISK_USAGE" -gt 60 ]; then
    echo "NOTICE: Disk usage is ${DISK_USAGE}%."
else
    echo "OK: Disk usage is ${DISK_USAGE}%."
fi

# String comparison
SERVICE_STATUS=$(systemctl is-active nginx 2>/dev/null)
if [ "$SERVICE_STATUS" = "active" ]; then
    echo "Nginx is running."
else
    echo "Nginx is NOT running."
fi
```

### Loops

```bash
#!/bin/bash

# For loop - iterate over list
for server in web01 web02 web03 db01; do
    echo "Pinging $server..."
    ping -c 1 "$server" > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo "  $server is UP"
    else
        echo "  $server is DOWN"
    fi
done

# For loop - iterate over range
for i in {1..5}; do
    echo "Creating user: student$i"
    # sudo useradd -m "student$i"
done

# While loop
COUNT=1
while [ $COUNT -le 5 ]; do
    echo "Attempt $COUNT"
    COUNT=$((COUNT + 1))
done

# Loop through files
for file in /var/log/*.log; do
    SIZE=$(du -sh "$file" | awk '{print $1}')
    echo "$file: $SIZE"
done
```

### Functions

```bash
#!/bin/bash

# Define a function
check_service() {
    local SERVICE_NAME=$1
    if systemctl is-active --quiet "$SERVICE_NAME"; then
        echo "[OK] $SERVICE_NAME is running"
        return 0
    else
        echo "[FAIL] $SERVICE_NAME is not running"
        return 1
    fi
}

# Call the function
check_service nginx
check_service mysql
check_service docker

# Function with return value
get_disk_usage() {
    df / | tail -1 | awk '{print $5}' | tr -d '%'
}

USAGE=$(get_disk_usage)
echo "Current disk usage: ${USAGE}%"
```

---

## Networking Basics

### Key Concepts

```
OSI Model (7 Layers):
┌─────────────────────────┐
│ 7. Application  (HTTP)  │  ← What users interact with
│ 6. Presentation (SSL)   │
│ 5. Session      (TLS)   │
│ 4. Transport    (TCP/UDP)│  ← Ports (80, 443, 22)
│ 3. Network      (IP)    │  ← IP addresses, routing
│ 2. Data Link    (MAC)   │  ← Switches
│ 1. Physical     (Cable) │  ← Hardware
└─────────────────────────┘

Common Ports:
  22  = SSH
  80  = HTTP
  443 = HTTPS
  3306 = MySQL
  5432 = PostgreSQL
  6379 = Redis
  27017 = MongoDB
  8080 = Common app port
```

### Network Commands

```bash
# Check IP address
ip addr show
# or
ifconfig

# Test connectivity
ping -c 4 google.com
# Output:
# PING google.com (142.250.80.46): 56 bytes
# 64 bytes from 142.250.80.46: icmp_seq=0 ttl=115 time=12.3 ms

# DNS lookup
nslookup google.com
# Output:
# Name:    google.com
# Address: 142.250.80.46

dig google.com          # Detailed DNS info

# Trace route to destination
traceroute google.com

# Check open ports on a host
netstat -tlnp           # Show listening ports
ss -tlnp                # Modern alternative to netstat

# Make HTTP requests
curl -v https://api.github.com    # Verbose HTTP request
curl -s https://api.github.com | head -20   # Silent mode
wget https://example.com/file.tar.gz        # Download a file

# Check if a port is open
nc -zv google.com 443
# Output: Connection to google.com 443 port [tcp/https] succeeded!
```

### DNS Configuration

```bash
# View current DNS
cat /etc/resolv.conf

# Edit hosts file (local DNS override)
sudo nano /etc/hosts

# Add entries:
# 192.168.56.10  myapp.local
# 192.168.56.11  db.local
```

---

## SSH Configuration

### Generate SSH Keys

```bash
# Generate a new SSH key pair
ssh-keygen -t ed25519 -C "student@devops"
# Press Enter for default location (~/.ssh/id_ed25519)
# Enter a passphrase (optional but recommended)

# View your public key
cat ~/.ssh/id_ed25519.pub
# Output: ssh-ed25519 AAAAC3NzaC1... student@devops
```

### Copy Key to Remote Server

```bash
# Method 1: Using ssh-copy-id
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.56.10

# Method 2: Manual copy
cat ~/.ssh/id_ed25519.pub | ssh user@192.168.56.10 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Now you can SSH without password
ssh user@192.168.56.10
```

### SSH Config File

```bash
# Create/edit SSH config
nano ~/.ssh/config
```

```
# ~/.ssh/config
Host devops-lab
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519
    Port 22

Host production
    HostName 203.0.113.50
    User deploy
    IdentityFile ~/.ssh/prod_key
    Port 2222

# Now you can simply run:
# ssh devops-lab
# ssh production
```

### Secure SSH Server

```bash
# Edit SSH server config
sudo nano /etc/ssh/sshd_config

# Recommended security settings:
# PermitRootLogin no              # Disable root login
# PasswordAuthentication no       # Only allow key-based auth
# PubkeyAuthentication yes        # Enable key auth
# Port 2222                       # Change default port
# MaxAuthTries 3                  # Limit login attempts

# Restart SSH service
sudo systemctl restart sshd
```

### SCP - Secure Copy

```bash
# Copy file to remote server
scp myfile.txt user@192.168.56.10:/home/user/

# Copy from remote to local
scp user@192.168.56.10:/var/log/app.log ./

# Copy entire directory
scp -r ./project/ user@192.168.56.10:/home/user/projects/
```

---

## Firewall Setup

### UFW (Uncomplicated Firewall) - Ubuntu

```bash
# Install and enable
sudo apt install -y ufw
sudo ufw enable

# Check status
sudo ufw status verbose

# Allow common services
sudo ufw allow ssh            # Port 22
sudo ufw allow 80/tcp         # HTTP
sudo ufw allow 443/tcp        # HTTPS
sudo ufw allow 8080/tcp       # Custom app port

# Allow from specific IP
sudo ufw allow from 192.168.1.100 to any port 22

# Deny a port
sudo ufw deny 3306            # Block MySQL from outside

# Delete a rule
sudo ufw delete allow 8080/tcp

# View rules with numbers
sudo ufw status numbered

# Reset all rules
sudo ufw reset
```

### iptables (Advanced)

```bash
# View current rules
sudo iptables -L -n -v

# Allow established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Drop all other incoming traffic
sudo iptables -A INPUT -j DROP

# Save rules (persist across reboots)
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## Cron Jobs

### Cron Syntax

```
* * * * * command_to_run
│ │ │ │ │
│ │ │ │ └── Day of Week (0-7, Sun=0 or 7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of Month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)

Examples:
  */5 * * * *     = Every 5 minutes
  0 * * * *       = Every hour
  0 2 * * *       = Daily at 2:00 AM
  0 0 * * 0       = Every Sunday at midnight
  0 9 * * 1-5     = Weekdays at 9:00 AM
  0 0 1 * *       = First day of every month
```

### Managing Cron Jobs

```bash
# Edit crontab for current user
crontab -e

# View crontab
crontab -l

# Example entries:
# Daily backup at 2 AM
0 2 * * * /home/student/scripts/backup.sh >> /var/log/backup.log 2>&1

# Check disk space every 30 minutes
*/30 * * * * /home/student/scripts/check_disk.sh

# Restart app every Monday at 3 AM
0 3 * * 1 sudo systemctl restart myapp

# Remove all cron jobs
crontab -r
```

---

## Projects

### Project 1: Automated Backup Script

```bash
#!/bin/bash
# File: backup.sh
# Description: Automated backup script with timestamping

BACKUP_SRC="/home/student/projects"
BACKUP_DEST="/home/student/backups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="backup_${TIMESTAMP}.tar.gz"
LOG_FILE="/var/log/backup.log"
RETENTION_DAYS=7

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DEST"

# Create compressed backup
echo "[$(date)] Starting backup..." | tee -a "$LOG_FILE"
tar -czf "${BACKUP_DEST}/${BACKUP_FILE}" "$BACKUP_SRC" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    SIZE=$(du -sh "${BACKUP_DEST}/${BACKUP_FILE}" | awk '{print $1}')
    echo "[$(date)] Backup successful: ${BACKUP_FILE} (${SIZE})" | tee -a "$LOG_FILE"
else
    echo "[$(date)] ERROR: Backup failed!" | tee -a "$LOG_FILE"
    exit 1
fi

# Remove backups older than retention period
echo "[$(date)] Cleaning up backups older than ${RETENTION_DAYS} days..." | tee -a "$LOG_FILE"
find "$BACKUP_DEST" -name "backup_*.tar.gz" -type f -mtime +${RETENTION_DAYS} -delete

echo "[$(date)] Backup process complete." | tee -a "$LOG_FILE"
```

```bash
# Make executable and test
chmod +x backup.sh
./backup.sh

# Add to crontab for daily execution at 2 AM
crontab -e
# Add: 0 2 * * * /home/student/scripts/backup.sh
```

### Project 2: Nginx Web Server Setup

```bash
#!/bin/bash
# File: setup_nginx.sh
# Description: Install and configure Nginx web server

# Update system
sudo apt update && sudo apt upgrade -y

# Install Nginx
sudo apt install -y nginx

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Create a custom website
sudo mkdir -p /var/www/mysite

sudo tee /var/www/mysite/index.html > /dev/null << 'HTML'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>DevOps Lab</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 50px; background: #1a1a2e; color: #eee; }
        h1 { color: #e94560; }
        .info { background: #16213e; padding: 20px; border-radius: 10px; display: inline-block; }
    </style>
</head>
<body>
    <h1>DevOps Lab Server</h1>
    <div class="info">
        <p>Server is running successfully!</p>
        <p>Hostname: <strong>HOSTNAME_PLACEHOLDER</strong></p>
        <p>Date: <strong>DATE_PLACEHOLDER</strong></p>
    </div>
</body>
</html>
HTML

# Replace placeholders
sudo sed -i "s/HOSTNAME_PLACEHOLDER/$(hostname)/" /var/www/mysite/index.html
sudo sed -i "s/DATE_PLACEHOLDER/$(date)/" /var/www/mysite/index.html

# Create Nginx server block
sudo tee /etc/nginx/sites-available/mysite > /dev/null << 'NGINX'
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/mysite_access.log;
    error_log /var/log/nginx/mysite_error.log;
}
NGINX

# Enable the site
sudo ln -sf /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# Test and reload
sudo nginx -t
sudo systemctl reload nginx

# Configure firewall
sudo ufw allow 'Nginx Full'

echo "Nginx setup complete! Visit http://$(hostname -I | awk '{print $1}')"
```

### Project 3: Disk Space Monitor

```bash
#!/bin/bash
# File: disk_monitor.sh
# Description: Monitor disk space and alert if threshold exceeded

THRESHOLD=80
EMAIL="admin@example.com"
LOG_FILE="/var/log/disk_monitor.log"

echo "========================================" | tee -a "$LOG_FILE"
echo "Disk Space Report - $(date)" | tee -a "$LOG_FILE"
echo "========================================" | tee -a "$LOG_FILE"

ALERT=false

while read -r line; do
    USAGE=$(echo "$line" | awk '{print $5}' | tr -d '%')
    MOUNT=$(echo "$line" | awk '{print $6}')
    AVAIL=$(echo "$line" | awk '{print $4}')

    if [ "$USAGE" -gt "$THRESHOLD" ]; then
        echo "[CRITICAL] $MOUNT is at ${USAGE}% (Available: $AVAIL)" | tee -a "$LOG_FILE"
        ALERT=true
    elif [ "$USAGE" -gt 60 ]; then
        echo "[WARNING]  $MOUNT is at ${USAGE}% (Available: $AVAIL)" | tee -a "$LOG_FILE"
    else
        echo "[OK]       $MOUNT is at ${USAGE}% (Available: $AVAIL)" | tee -a "$LOG_FILE"
    fi
done < <(df -h | grep '^/dev/')

if [ "$ALERT" = true ]; then
    echo "" | tee -a "$LOG_FILE"
    echo "!!! ALERT: One or more partitions exceeded ${THRESHOLD}% usage !!!" | tee -a "$LOG_FILE"
    # Uncomment to send email alert:
    # echo "Disk space critical on $(hostname)" | mail -s "Disk Alert" "$EMAIL"
fi

echo "" | tee -a "$LOG_FILE"

# Top 10 largest directories
echo "Top 10 Largest Directories in /:" | tee -a "$LOG_FILE"
du -sh /* 2>/dev/null | sort -rh | head -10 | tee -a "$LOG_FILE"
```

```bash
# Make executable and schedule
chmod +x disk_monitor.sh
./disk_monitor.sh

# Run every 30 minutes via cron
crontab -e
# Add: */30 * * * * /home/student/scripts/disk_monitor.sh
```

---

## Quick Reference Cheat Sheet

| Command | Description |
|---------|-------------|
| `ls -la` | List all files with details |
| `chmod 755 file` | Set permissions rwxr-xr-x |
| `chown user:group file` | Change ownership |
| `ps aux \| grep name` | Find a process |
| `systemctl status service` | Check service status |
| `df -h` | Disk space |
| `free -h` | Memory usage |
| `grep -r "text" /path` | Search recursively |
| `find . -name "*.log"` | Find files by name |
| `ssh-keygen -t ed25519` | Generate SSH key |
| `ufw allow 80/tcp` | Open firewall port |
| `crontab -e` | Edit cron jobs |
