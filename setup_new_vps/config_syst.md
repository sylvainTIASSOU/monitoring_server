## Features:
- 🔒 Secure SSH configuration
- 🐍 Python environment with deadsnakes PPA
- 🚀 Nginx with UI management
- 🐳 Docker and Docker Compose
- 🔑 SSL/TLS with Let's Encrypt
- 📊 MongoDB Atlas integration
- 💾 Swap and system optimization
- 🔄 Auto-renewal and maintenance scripts
- ⏰ Custom MOTD with system stats
- 📝 Memory monitoring and management

Perfect for developers setting up a new VPS or standardizing their server configuration. Includes detailed step-by-step instructions and best practices for security and performance.

## Usage:
1. Follow sections in order
2. Copy-paste commands as needed
3. Modify configurations according to your needs

## Requirements:
- Ubuntu Server (Latest LTS recommended)
- Root or sudo access
- Basic command line knowledge

## Post-Setup:
- Nginx UI accessible at port 9000
- Python versions: 3.8 through 3.12
- UV package manager for faster Python package installation
- Optimized for both web hosting and development

---
### Table of Contents
1.  [Initial System Setup](#1-initial-system-setup)
2.  [Security Configurations](#2-security-configurations)
3.  [System Optimizations](#3-system-optimizations)
4.  [Development Environment Setup](#4-development-environment-setup)
5.  [Web Server Setup](#5-web-server-setup)
6.  [Database Setup](#6-database-setup)
7.  [Docker Setup](#7-docker-setup)
8.  [SSL Configuration](#8-ssl-configuration)
9.  [Monitoring and Maintenance](#9-monitoring-and-maintenance)

---

## 1. Initial System Setup

**Context:** This section focuses on the preliminary steps required to set up your VPS. It will cover creating a new user, setting up SSH for secure access, and updating the system to ensure all packages are up to date.

### Create New User

**Explanation**: Creating a new user with sudo privileges enhances security by avoiding direct usage of the root user.

```bash
# Create a new user called satya
sudo adduser satya

# Add user to the sudo group to grant administrative privileges
sudo usermod -aG sudo satya

# Switch to the new user
su - satya
```

### SSH Setup

**Explanation**: Setting up SSH keys allows for passwordless login, which is more secure than using passwords.

```bash
# On local machine, generate a new SSH key pair if you don't already have one. Replace "your@email.com" with your email address.
ssh-keygen -t ed25519 -C "your@email.com"

# Copy the public key to the server
ssh-copy-id satya@server_ip

# On the server, ensure that the .ssh directory and its contents have proper permissions.
mkdir -p ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*
```

### Configure SSH

**Explanation**: This configures the SSH daemon for enhanced security and prevents certain vulnerabilities.

```bash
# Edit the SSH configuration file
sudo nano /etc/ssh/sshd_config
```
```
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
```
```bash
# Restart the SSH service for the changes to take effect
sudo systemctl restart sshd
```

### System Updates

**Explanation:** Ensures all installed packages are up-to-date and includes some useful utilities.

```bash
# Update the package lists
sudo apt update

# Upgrade installed packages
sudo apt upgrade -y

# Install commonly used utilities
sudo apt install curl wget git htop neofetch net-tools -y
```

### MOTD (Message of the Day)

**Explanation**: This customizes the message displayed upon SSH login with some helpful system information.

```bash
# Install figlet and lolcat
sudo apt install figlet lolcat -y

# Create a custom MOTD script
sudo nano /etc/update-motd.d/00-custom
```

```bash
#!/bin/bash
echo
figlet "DevH Server" | lolcat
echo
echo "Welcome to $(hostname)"
echo "System Info: $(lsb_release -ds)"
echo "Kernel: $(uname -r)"
echo
echo "Memory Usage: $(free -h | awk '/^Mem:/ {print $3 "/" $2}')"
echo "Disk Usage: $(df -h / | awk 'NR==2 {print $3 "/" $2}')"
echo "Load Average: $(cat /proc/loadavg | awk '{print $1, $2, $3}')"
echo
if [ -f /var/run/reboot-required ]; then
    echo "System restart required!"
fi
echo
```
```bash
# Make the MOTD script executable
sudo chmod +x /etc/update-motd.d/00-custom
```

---

## 2. Security Configurations

**Context**: This section is dedicated to security enhancements for your VPS. It will cover firewall setup, SSH hardening, and general security recommendations.

### Firewall Setup

**Explanation:** Setting up a firewall is crucial for protecting the server from unauthorized access. `ufw` is used here for ease of use.

```bash
# Install ufw
sudo apt install ufw -y

# Allow SSH connections
sudo ufw allow ssh

# Allow HTTP connections
sudo ufw allow http

# Allow HTTPS connections
sudo ufw allow https

# Enable the firewall
sudo ufw enable

# Check firewall status
sudo ufw status
```

### SSH Hardening

**Explanation:** SSH is a common target for attacks, so these steps will help further secure it.

```bash
# Edit the SSH configuration file.
sudo nano /etc/ssh/sshd_config
```

```
# Disable password-based authentication.
PasswordAuthentication no
# Specify acceptable ciphers and MACs.
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
```
```bash
# Restart the SSH service to apply the changes
sudo systemctl restart sshd
```

---

## 3. System Optimizations

**Context:** This section focuses on improving the performance and efficiency of your VPS. It will include swap space configuration and kernel parameter tuning.

### Swap Setup

**Explanation:** Swap space is used when the system runs out of RAM. It's created here and activated.

```bash
# Create a 4GB swap file
sudo fallocate -l 4G /swapfile

# Set appropriate permissions
sudo chmod 600 /swapfile

# Create swap space
sudo mkswap /swapfile

# Activate swap space
sudo swapon /swapfile

# Add the swap file to fstab to make it persistent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### System Tuning

**Explanation:** Kernel tuning optimizes network and memory usage, and improves performance.

```bash
# Edit the sysctl configuration file
sudo nano /etc/sysctl.conf
```
```
# Memory Management
vm.swappiness=10
vm.vfs_cache_pressure=50
vm.page-cluster=0
vm.dirty_ratio=10
vm.dirty_background_ratio=5

# Network Optimization
net.core.somaxconn=65536
net.ipv4.tcp_max_syn_backlog=4096
net.core.netdev_max_backlog=4096
net.ipv4.tcp_fastopen=3
```
```bash
# Apply sysctl changes
sudo sysctl -p
```

---

## 4. Development Environment Setup

**Context:** This section sets up the Python development environment on your VPS, including the installation of multiple Python versions, development packages, and the `uv` package installer.

### Install Python Versions (Deadsnakes PPA)

**Explanation:** The Deadsnakes PPA provides multiple versions of Python for development.

```bash
# Add the Deadsnakes PPA
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update

# Install various Python versions
sudo apt install python3.8 python3.9 python3.10 python3.11 python3.12 -y

# Install development headers for python 3.11
sudo apt install python3.11-dev python3.11-venv -y

# Install pip
curl -sS https://bootstrap.pypa.io/get-pip.py | python3.11
```

### Install uv (Fast Python Package Installer)

**Explanation:** uv is a fast alternative to pip for managing Python packages.

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Add uv to the PATH. The user is expected to be the satya user.
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc

# Refresh the environment
source ~/.bashrc
```

---

## 5. Web Server Setup

**Context:** This section covers the installation and basic configuration of Nginx as a web server, as well as the installation and setup of Nginx UI for easier management of your web server.

### Install Nginx

```bash
# Install Nginx
sudo apt install nginx -y
```

### Nginx Configuration

```bash
# Edit the Nginx configuration file. The default is normally sufficient.
sudo nano /etc/nginx/nginx.conf
```
**Note:** The default Nginx config should work. If you have custom requirements, update this file appropriately.

### Install Nginx UI

```bash
# Download and run the Nginx UI installation script
curl -L -s https://raw.githubusercontent.com/0xJacky/nginx-ui/main/install.sh -o nginx-ui-install.sh
chmod +x nginx-ui-install.sh
sudo ./nginx-ui-install.sh install
rm nginx-ui-install.sh
```

### Configure Nginx UI

```bash
# Edit the Nginx UI configuration file
sudo nano /usr/local/etc/nginx-ui/app.ini
```
```ini
[server]
http_port = 9000
debug = false
single_process = true

[database]
type = sqlite3
max_idle_conns = 2
max_open_conns = 5

[log]
level = warn
```

---

## 6. Database Setup

**Context:** This section covers the setup of MongoDB Atlas, including the installation of the MongoDB shell and database tools. It presumes an external database is being used rather than one installed directly on the server.

### Install MongoDB Shell and Tools
**Explanation**: This section explains how to install the MongoDB shell and other relevant tools

```bash
# Add MongoDB GPG key
curl -fsSL https://pgp.mongodb.com/server-6.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg \
   --dearmor

# Add MongoDB repository
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

# Update the package list
sudo apt-get update

# Install the MongoDB shell
sudo apt-get install -y mongodb-mongosh

# Install MongoDB database tools
sudo apt-get install -y mongodb-database-tools
```

---

## 7. Docker Setup

**Context**: This section details the installation of Docker and Docker Compose, which can be used for containerization of services.

### Install Docker

```bash
# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add user to the docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl enable docker
sudo systemctl start docker
```

### Install Docker Compose

```bash
# Install using apt
sudo apt-get install docker-compose-plugin -y

# Verify installation
docker compose version
```

---

## 8. SSL Configuration

**Context:** This section covers how to install and configure SSL certificates for your domains using Let's Encrypt and Certbot.

### Install Certbot

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y
```

### Obtain SSL Certificate

```bash
# For a single domain
sudo certbot --nginx -d example.com

# For multiple domains
sudo certbot --nginx -d example.com -d www.example.com

# For wildcard certificate
sudo certbot certonly --manual \
  --preferred-challenges=dns \
  --email your@email.com \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos \
  -d *.example.com
```

### Auto-renewal Setup

```bash
# Test auto-renewal
sudo certbot renew --dry-run

# Verify the auto-renewal timer created by certbot
systemctl list-timers
```

---

## 9. Monitoring and Maintenance

**Context:** This section details how to set up system monitoring and includes regular maintenance tasks for your VPS, including memory monitoring and system updates.

### Memory Monitoring Script

```bash
# Create memory check script
sudo nano /usr/local/bin/memcheck.sh
```
```bash
#!/bin/bash

THRESHOLD=85
MEMORY_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')

if [ $MEMORY_USAGE -gt $THRESHOLD ]; then
    sync; echo 3 > /proc/sys/vm/drop_caches
    swapoff -a && swapon -a
    systemctl restart nginx-ui
fi
```
```bash
# Make the script executable
sudo chmod +x /usr/local/bin/memcheck.sh
```

### Add to Crontab
**Explanation:** This sets up a cron job to run the memory monitoring script regularly.

```bash
# Edit the crontab file
sudo nano /etc/crontab
```
Add:
```
*/5 * * * * root /usr/local/bin/memcheck.sh > /dev/null 2>&1
```

### Final Steps

**Explanation:** These final steps include important considerations for completing the server setup, such as setting the timezone and creating backups.

### Set Timezone
```bash
sudo timedatectl set-timezone Asia/Kolkata
```

### Create Backup Directory
```bash
mkdir -p ~/backups
```

### Setup Completed
-   Access Nginx UI: `https://your-server-ip:9000`
-   Default credentials: `admin/admin`
-   Change default passwords
-   Configure firewall rules
-   Set up regular backups

## Regular Maintenance Tasks
1.  System Updates:
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2.  Docker Cleanup:
    ```bash
    docker system prune -af
    ```

3.  Log Rotation:
    ```bash
    sudo logrotate -f /etc/logrotate.conf
    ```

4.  Certificate Renewal:
    ```bash
    sudo certbot renew
    ```

---

**Note:**

*   You should have basic familiarity with Linux commands and command-line interfaces.
*   You should have a domain name configured to point to the server.
*   You should have a local machine with SSH capabilities.
*   You should have a basic understanding of networking and web servers.
*   My target is an Ubuntu server!
*   I included Explanations and Context for each section for better understanding

> What to do if you stuck? Go, get some help!
