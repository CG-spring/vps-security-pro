# VPS Security Pro - Advanced Security Hardening Guide

[![License](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/CG-spring/vps-security-pro.svg?style=flat-square)](https://github.com/CG-spring/vps-security-pro/stargazers)

> Enterprise-level security hardening, firewall configuration and advanced protection strategies for Linux VPS
>
> Continuously Updated | Last Updated: 2026-08-12

**中文** | **[English](README_EN.md)**

---

## Table of Contents

- [Why Advanced Security Hardening?](#why-advanced-security-hardening)
- [SSH Advanced Hardening](#ssh-advanced-hardening)
- [Enterprise Firewall Configuration](#enterprise-firewall-configuration)
- [fail2ban Advanced Configuration](#fail2ban-advanced-configuration)
- [DDoS Protection Solutions](#ddos-protection-solutions)
- [Security Monitoring and Alerts](#security-monitoring-and-alerts)
- [Complete Security Scripts](#complete-security-scripts)
- [FAQ](#faq)
- [Recommended VPS Providers](#recommended-vps-providers)

---

## Why Advanced Security Hardening?

A VPS on the public internet faces constant threats — automated botnets scanning for open ports, brute-force attacks on SSH, DDoS floods, and exploitation of unpatched services. The default configuration of most Linux distributions prioritizes usability over security, leaving new VPS deployments dangerously exposed within minutes of going online.

### Standard vs. Enterprise-Level Protection

| Security Layer | Standard Approach | Enterprise Approach |
|---------------|-------------------|---------------------|
| SSH Hardening | Change default port | Key-based auth + IP whitelist + 2FA |
| Firewall | Basic port rules | Complex rules + rate limiting + geo-blocking |
| Intrusion Detection | None | Behavioral analysis + real-time alerts |
| Log Analysis | Manual review | Centralized logging + automated analysis |
| Backup | Manual snapshots | Automated + offsite + geo-redundant |
| Updates | Manual | Automated unattended security patches |

### Common Attack Vectors on VPS

| Attack Type | Risk Level | Impact | Mitigation |
|-------------|-----------|--------|-----------|
| SSH Brute Force | 🔴 Critical | Server takeover, data theft | Key auth + fail2ban + IP whitelist |
| DDoS Attack | 🔴 Critical | Service unavailable | Traffic scrubbing + rate limiting |
| Port Scanning | 🟡 High | Reconnaissance for targeted attacks | Firewall stealth + minimal exposure |
| Zero-Day Exploit | 🔴 Critical | Unknown risk | Minimize attack surface + rapid patching |
| Supply Chain Attack | 🟡 High | Backdoored packages | Use official repos only + GPG verification |
| Privilege Escalation | 🔴 Critical | Root access from user compromise | Principle of least privilege + AppArmor |

### Attack Statistics You Should Know

- **SSH brute force attempts**: A typical VPS will receive 100-500 attempted logins per day by automated bots
- **Time to first probe**: Most VPS instances are scanned within 15 minutes of going online
- **Compromised servers**: 70% of compromised VPS instances are used for cryptocurrency mining
- **Average breach cost**: $3.86 million globally (IBM 2024 report)

This guide covers enterprise-grade hardening techniques that go far beyond simply changing your SSH port.

---

## SSH Advanced Hardening

SSH is the primary entry point to your VPS and the most targeted service. A compromised SSH daemon typically means full server takeover.

### 1. Two-Factor Authentication (2FA)

Password-based authentication is fundamentally insecure — it's susceptible to phishing, brute force, and credential stuffing. TOTP-based 2FA adds a time-based one-time password layer that cannot be replayed.

```bash
# Install Google Authenticator PAM module
# Debian/Ubuntu
apt update && apt install -y libpam-google-authenticator

# RHEL/CentOS/AlmaLinux
yum install -y google-authenticator

# Run as the user who needs 2FA (each user configures their own)
su - your_username
google-authenticator

# Answer prompts:
# Make tokens time-based? → Yes
# Update your ~/.google_authenticator file? → Yes
# Disallow multiple uses of the same authentication? → Yes
# Enable rate limiting? → Yes
# Allow 30-second window? → No (default 1:30 = 3 window is safer)

# Configure PAM to require Google Authenticator
vim /etc/pam.d/sshd
# Add the following line at the end:
auth required pam_google_authenticator.so

# Configure SSH daemon
vim /etc/ssh/sshd_config

# Ensure these settings are present:
Port 22022                    # Non-standard port
Protocol 2                    # SSH protocol version 2 only
PermitRootLogin no            # Never allow root login
PasswordAuthentication no     # Disable password auth entirely
PermitEmptyPasswords no       # Reject empty passwords
ClientAliveInterval 300       # Disconnect idle sessions after 5 min
ClientAliveCountMax 2         # Allow max 2 missed keepalives
MaxAuthTries 3                # Fail after 3 failed attempts
LoginGraceTime 60             # Grace period for auth

# Enable challenge-response authentication
ChallengeResponseAuthentication yes
AuthenticationMethods keyboard-interactive

# Restart SSH service
systemctl restart sshd

# IMPORTANT: Keep your current SSH session open!
# Test the new connection in a separate terminal before closing
```

**Why disable password authentication entirely?** Even with fail2ban, brute-force attacks can occasionally succeed if passwords are weak. Key-based authentication is mathematically resistant to brute-force attacks — an RSA-4096 key would take longer than the age of the universe to crack by brute force.

### 2. SSH Key + Passphrase Best Practices

```bash
# Generate a strong ED25519 key (preferred over RSA)
ssh-keygen -t ed25519 -f ~/.ssh/vps_master -C "vps-master-key-$(date +%Y%m)"

# The ED25519 algorithm is:
# - Faster to authenticate with
# - Smaller key size (256 bits vs 4096 for RSA)
# - Resistant to certain side-channel attacks
# - Modern and well-vetted

# Store your private key in a password manager (e.g., Bitwarden, 1Password)
# The passphrase protects your key if the file is stolen

# Upload public key to VPS
ssh-copy-id -i ~/.ssh/vps_master.pub admin@your-vps-ip

# Verify the key works before disabling passwords
ssh -i ~/.ssh/vps_master -p 22022 admin@your-vps-ip

# Create SSH config for convenience
vim ~/.ssh/config

Host vps
    HostName your-vps-ip
    User admin
    Port 22022
    IdentityFile ~/.ssh/vps_master
    IdentitiesOnly yes
    AddKeysToAgent yes          # Add key to agent on first use
    ForwardAgent no             # Don't forward agent to remote
    ServerAliveInterval 120     # Keep connection alive
    ServerAliveCountMax 3

# Now you can connect simply with:
ssh vps
```

### 3. IP Whitelist + Geo-Blocking

The most effective SSH protection is allowing only known IP addresses. If you have a static IP or use a VPN, this eliminates 99.9% of SSH attacks.

```bash
# Allow only specific IPs for SSH (hosts.allow takes precedence)
vim /etc/hosts.allow

# Add your specific IP addresses
sshd: 203.0.113.42 :allow      # Your home IP
sshd: 198.51.100.0/24 :allow   # Your office network range
sshd: [2001:db8::1] :allow     # IPv6 address

vim /etc/hosts.deny

sshd: ALL :deny                # Deny everyone else

# For dynamic IPs, consider a VPN with a static exit IP
# Or use a jump host / bastion server
```

Geo-blocking with UFW (for countries you don't need access from):

```bash
# List countries by ISO code you want to block
# China: CN, Russia: RU, North Korea: KP, Iran: IR, etc.

# Using UFW with country blocks
ufw deny from 1.0.0.0/8 to any port 22 comment 'Block region'
ufw deny from 5.0.0.0/8 to any port 22 comment 'Block region'
# (Note: UFW doesn't natively support country-level blocking;
#  use iptables with ipset for production geo-blocking)

# Using ipset for efficient geo-blocking
apt install -y ipset ipset-persistent
ipset create blocked_countries hash:net
# Add IP ranges for countries to block
ipset add blocked_countries 1.0.0.0/8
ipset add blocked_countries 5.0.0.0/8

# Apply via iptables
iptables -A INPUT -p tcp --dport 22 -m set --match-set blocked_countries src -j DROP
```

---

## Enterprise Firewall Configuration

A properly configured firewall is your first line of defense. Default-deny policies mean only explicitly allowed traffic passes.

### 1. UFW Advanced Rules

UFW (Uncomplicated Firewall) provides a user-friendly interface to iptables while retaining full power.

```bash
# Install UFW
apt update && apt install -y ufw

# Set default policies (deny everything not explicitly allowed)
ufw default deny incoming
ufw default allow outgoing
ufw default deny routed          # If this VPS is not a router

# Allow SSH from your IP (CRITICAL - do this before enabling!)
# Replace 1.2.3.4 with YOUR actual IP address
ufw allow from 1.2.3.4 to any port 22 proto tcp comment 'My home IP'

# Rate limiting for SSH (blocks IPs with >6 connections per 30 seconds)
ufw limit from any to any port 22 proto tcp comment 'SSH rate limit'

# Allow web traffic
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

# Allow common services as needed
ufw allow 3306/tcp comment 'MySQL - internal only'
ufw allow 5432/tcp comment 'PostgreSQL - internal only'
ufw allow 6379/tcp comment 'Redis - internal only'
ufw allow 27017/tcp comment 'MongoDB - internal only'

# Allow monitoring agents
ufw allow from 10.0.0.0/24 to any port 9100 proto tcp comment 'Prometheus node exporter'

# View current rules
ufw status numbered

# Enable the firewall (do this LAST after adding your SSH rule!)
ufw --force enable

# Check status
ufw status verbose
```

### 2. iptables Advanced Rules

For more granular control, use iptables directly. Here's a comprehensive hardening script:

```bash
#!/bin/bash
# enterprise_firewall.sh - Advanced iptables hardening

set -e

echo "Applying enterprise firewall rules..."

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback traffic
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established and related connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH with connection tracking and rate limiting
# New SSH connections limited to 3 per minute per IP
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW     -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW     -m recent --update --seconds 60 --hitcount 4 --rttl --name SSH -j DROP
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# Prevent SYN flood attacks
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# Prevent ping flood (ping of death)
iptables -A INPUT -p icmp --icmp-type echo-request     -m limit --limit 1/s --limit-burst 4 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Drop invalid packets
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Block common scan patterns
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP        # Null scan
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP         # XMAS scan
iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP # SYN-FIN scan

# Allow web traffic
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow DNS (both directions)
iptables -A INPUT -p udp --dport 53 -j ACCEPT
iptables -A INPUT -p tcp --dport 53 -j ACCEPT

# Log dropped packets (use with caution - can fill logs)
# iptables -A INPUT -m limit --limit 5/min -j LOG #     --log-prefix "iptables-dropped: " --log-level 4

echo "Firewall rules applied successfully."

# Save rules (Debian/Ubuntu)
if command -v netfilter-persistent &> /dev/null; then
    netfilter-persistent save
elif command -v iptables-persistent &> /dev/null; then
    iptables-save > /etc/iptables/rules.v4
else
    # Manual save
    iptables-save > /etc/iptables/rules.v4
    echo "#!/bin/sh" > /etc/network/if-pre-up.d/iptables
    echo "iptables-restore < /etc/iptables/rules.v4" >> /etc/network/if-pre-up.d/iptables
    chmod +x /etc/network/if-pre-up.d/iptables
fi

echo "Rules saved to /etc/iptables/rules.v4"
```

### 3. Automated IP Blocking Script

This script analyzes auth logs and automatically blocks attacking IPs:

```bash
#!/bin/bash
# auto_block.sh - Automatically block brute-force attackers

LOG_FILE="/var/log/auth.log"
BLOCKED_IPS="/tmp/blocked_ips.txt"
THRESHOLD=10           # Failed attempts before blocking
TIME_WINDOW="1 hour"   # Time window to analyze

# Initialize blocked IPs file
touch "$BLOCKED_IPS"

# Get failed login IPs from the past hour
FAILED_IPS=$(grep "Failed password" "$LOG_FILE" |     grep "$(date -d '1 hour ago' +'%b %d')" |     awk '{print $11}' |     sort | uniq -c |     awk -v thresh="$THRESHOLD" '$1 >= thresh {print $2}')

for IP in $FAILED_IPS; do
    # Validate IP format
    if [[ ! "$IP" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
        continue
    fi

    # Don't block private IPs
    if [[ "$IP" =~ ^(10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.) ]]; then
        continue
    fi

    # Check if already blocked
    if ! grep -qF "$IP" "$BLOCKED_IPS"; then
        echo "$IP" >> "$BLOCKED_IPS"
        ufw insert 1 deny from "$IP" to any comment "Auto-blocked: brute force" 2>/dev/null
        echo "$(date '+%Y-%m-%d %H:%M:%S') Blocked $IP (failed attempts: $(grep -c "$IP" <<< "$FAILED_IPS"))"
    fi
done

# Cleanup old IPs (unblock after 7 days)
# This would require tracking block timestamps
```

Run this script via cron every 10 minutes:
```bash
crontab -e
# */10 * * * * /path/to/auto_block.sh >> /var/log/auto_block.log 2>&1
```

---

## fail2ban Advanced Configuration

fail2ban monitors log files and automatically creates firewall rules to ban IPs showing malicious behavior. It's your automated security guard.

### 1. Multi-Jail Configuration

```bash
# Install fail2ban
apt update && apt install -y fail2ban

# Create local configuration (don't edit .conf files directly)
vim /etc/fail2ban/jail.local

[DEFAULT]
# Ban duration: 1 hour for first offense, 1 day for repeat offenders
bantime = 3600
# Time window: 10 minutes
findtime = 600
# Max attempts before ban
maxretry = 3
# Email notifications
destemail = admin@your-domain.com
sender = fail2ban@your-domain.com
# action = %(action_mwl)s means: ban + email with whois info + log lines
action = %(action_mwl)s
# Log target
logtarget = /var/log/fail2ban.log

# SSH jail - highest priority
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400           # 1 day ban for SSH attacks
findtime = 600

# Using the aggressive filter for sshd (better than default)
backend = auto

# Nginx HTTP auth jail
[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 5
bantime = 7200
findtime = 600

# WordPress login protection
[wp-login]
enabled = true
port = http,https
filter = wp-login
logpath = /var/log/nginx/access.log
maxretry = 5
bantime = 7200
findtime = 600
# Custom log regex for WordPress (if not using standard wp-login.php)
logencoding = utf-8

# Apache2 brute force
[apache-auth]
enabled = true
port = http,https
filter = apache-auth
logpath = /var/log/apache2/error.log
maxretry = 6
bantime = 3600

# vsFTPd protection
[vsftpd]
enabled = true
port = ftp,ftp-data,21,20,990
filter = vsftpd
logpath = /var/log/vsftpd.log
maxretry = 5
bantime = 3600

# Postfix SASL (mail server auth)
[postfix-sasl]
enabled = true
port = smtp,ssmtp,submission
filter = postfix-sasl
logpath = /var/log/mail.log
maxretry = 3
bantime = 86400

# Custom SSH brute-force detection with better regex
[sshd-auth]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
findtime = 300
# Use aggressive mode
action = iptables-allports[name=sshd-auth, protocol=all]

# Save and restart
systemctl restart fail2ban

# Verify status
fail2ban-client status
fail2ban-client status sshd
```

### 2. Custom fail2ban Filters

For specialized applications, create custom filters:

```bash
# WordPress login filter
vim /etc/fail2ban/filter.d/wp-login.conf

[Definition]
failregex = ^<HOST> - - \[.*\] "POST /wp-login.php
ignoreregex =

# Proxmox VE login
vim /etc/fail2ban/filter.d/proxmox.conf

[Definition]
failregex = pve-hook-helper: authentication failure;.*rhost=<HOST>
ignoreregex =

# Check filter regex before enabling
fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/wp-login.conf
```

### 3. Monitoring fail2ban Status

```bash
# View all active bans
fail2ban-client banned

# Check specific jail
fail2ban-client status sshd

# Unban an IP (e.g., if you locked yourself out)
fail2ban-client set sshd unbanip 1.2.3.4

# Temporarily pause a jail
fail2ban-client set sshd pause

# Add an IP to ignore list (whitelist)
vim /etc/fail2ban/jail.local

[DEFAULT]
ignorecommand =

# Add to jail.local under [DEFAULT]
# These IPs will never be banned
ignoreip = 127.0.0.1/8 1.2.3.4 10.0.0.0/8

# View fail2ban log for recent activity
tail -50 /var/log/fail2ban.log
```

---

## DDoS Protection Solutions

DDoS attacks flood your server with more traffic than it can handle, making it unreachable. Layer 7 (application layer) attacks are particularly dangerous because they mimic legitimate traffic.

### 1. Application Layer Protection (Nginx)

```nginx
# /etc/nginx/nginx.conf or /etc/nginx/sites-available/default

# Connection zone - limits concurrent connections per client IP
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

server {
    listen 80;
    server_name your-domain.com;

    # Limit connections per IP
    limit_conn addr 10;

    # Limit requests per second (burst of 20, then queuing)
    limit_req zone=one burst=20 nodelay;

    # Limit request body size (prevent large file uploads)
    client_max_body_size 10M;

    # Timeouts - prevent slow-read attacks
    client_body_timeout 10s;
    client_header_timeout 10s;
    send_timeout 10s;

    # Hide nginx version
    server_tokens off;

    # Hotlink protection
    location ~* \.(jpg|png|gif|css|js)$ {
        valid_referers none blocked your-domain.com;
        if ($invalid_referer) {
            return 403;
        }
    }

    # Block common exploit attempts
    location ~ /\.(svn|git|hg|bzr|cvs) {
        deny all;
    }

    # WordPress specific - block XML-RPC if not needed
    location = /xmlrpc.php {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

### 2. System-Level DDoS Mitigation

```bash
# Add to /etc/sysctl.conf for DDoS resilience

# === SYN Flood Protection ===
net.ipv4.tcp_syncookies = 1              # Enable SYN cookies
net.ipv4.tcp_syn_retries = 2             # Reduce SYN retries
net.ipv4.tcp_synack_retries = 2          # Reduce SYN-ACK retries
net.ipv4.tcp_max_syn_backlog = 8192      # Larger SYN queue

# === Connection Limits ===
net.ipv4.ip_local_port_range = 2000 65000
net.ipv4.tcp_max_tw_buckets = 2000       # Reduce TIME_WAIT buckets
net.ipv4.tcp_fin_timeout = 15            # Faster FIN timeout
net.ipv4.tcp_tw_reuse = 1                # Reuse TIME_WAIT sockets
net.ipv4.tcp_max_orphans = 262144        # Max orphaned sockets
net.ipv4.tcp_keepalive_time = 300        # Keepalive interval

# === ICMP Protection ===
net.ipv4.icmp_echo_ignore_broadcasts = 1 # Ignore broadcast pings
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.icmp_ratelimit = 50             # Limit ICMP rate

# === IP Spoofing Protection ===
net.ipv4.conf.all.rp_filter = 1          # Enable source validation
net.ipv4.conf.default.rp_filter = 1

# === TCP Hardening ===
net.ipv4.conf.all.accept_redirects = 0   # Reject ICMP redirects
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Apply changes
sysctl -p

# Verify settings
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.conf.all.rp_filter
```

### 3. Connection Limiting with iptables

```bash
# Limit concurrent connections per IP to port 80/443
iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 100 --connlimit-mask 32 -j DROP
iptables -A INPUT -p tcp --dport 443 -m connlimit --connlimit-above 100 --connlimit-mask 32 -j DROP

# Limit new connections per IP per minute
iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m recent --update --seconds 60 --hitcount 30 -j DROP

# Save
iptables-save > /etc/iptables/rules.v4
```

---

## Security Monitoring and Alerts

Proactive monitoring lets you detect and respond to attacks before they succeed.

### 1. Real-Time Security Dashboard Script

```bash
#!/bin/bash
# security_dashboard.sh - Real-time VPS security status

echo "=============================================="
echo "  VPS Security Monitoring Dashboard"
echo "  $(date '+%Y-%m-%d %H:%M:%S %Z')"
echo "=============================================="
echo ""

# --- SSH Security Status ---
echo "[SSH Security]"
echo "-----------------------------"
echo "  Current SSH sessions: $(who | wc -l)"
echo "  SSH failed logins (today): $(grep "$(date +%b\ %d)" /var/log/auth.log 2>/dev/null | grep -c "Failed password")"
echo "  fail2ban banned IPs: $(fail2ban-client banned 2>/dev/null | tail -n +2 | wc -l)"
echo ""

# --- Network Status ---
echo "[Network Status]"
echo "-----------------------------"
echo "  Total TCP connections: $(netstat -an | grep tcp | wc -l)"
echo "  Total UDP connections: $(netstat -an | grep udp | wc -l)"
echo "  ESTABLISHED: $(netstat -an | grep ESTABLISHED | wc -l)"
echo "  TIME_WAIT: $(netstat -an | grep TIME_WAIT | wc -l)"
echo ""

# --- Listening Ports (Security Check) ---
echo "[Listening Ports - Security Review]"
echo "-----------------------------"
netstat -tuln | awk 'NR>2 {print "  " $1, $4, $6}' | grep LISTEN
echo ""

# --- Firewall Status ---
echo "[Firewall Status]"
echo "-----------------------------"
ufw status verbose | head -8
echo ""

# --- System Resource Check ---
echo "[System Resources]"
echo "-----------------------------"
echo "  Memory usage: $(free -m | awk 'NR==2{printf "%.1f%%", $3/$2*100}')"
echo "  Disk usage: $(df -h / | awk 'NR==2{print $5}')"
echo "  Load average: $(uptime | awk -F'load average:' '{print $2}')"
echo ""

# --- Recent Security Events ---
echo "[Recent Security Events]"
echo "-----------------------------"
tail -10 /var/log/auth.log 2>/dev/null | grep -i "failed\|break-in\|illegal\|banned" || echo "  (No recent events)"
echo ""

# --- Fail2ban Recent Bans ---
echo "[Recent fail2ban Bans]"
echo "-----------------------------"
journalctl -u fail2ban -n 15 --no-pager 2>/dev/null | grep -i "ban\|unban" | tail -5 || echo "  (No recent bans)"
echo ""

echo "=============================================="
echo "  Next check: Run again or set up cron"
echo "=============================================="
```

### 2. Telegram Alert Script

```bash
#!/bin/bash
# telegram_alert.sh - Send security alerts via Telegram Bot

TELEGRAM_BOT_TOKEN="YOUR_BOT_TOKEN_HERE"
TELEGRAM_CHAT_ID="YOUR_CHAT_ID_HERE"

send_telegram() {
    local message="$1"
    local parse_mode="HTML"
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage"         -d "chat_id=${TELEGRAM_CHAT_ID}"         -d "text=${message}"         -d "parse_mode=${parse_mode}"         -d "disable_notification=false"
}

# Example: SSH brute force alert
send_telegram "🛡️ <b>VPS Security Alert</b>%0A%0A🔴 <b>SSH Brute Force Detected</b>%0AIP: <code>192.168.1.100</code>%0AAttempts: <code>50+</code>%0AAction: <b>Auto-blocked</b>%0A%0ATime: $(date '+%Y-%m-%d %H:%M:%S')"

# Example: New root login alert
send_telegram "🛡️ <b>VPS Security Alert</b>%0A%0A🟡 <b>Root Login Detected</b>%0AIP: <code>1.2.3.4</code>%0ATime: $(date '+%Y-%m-%d %H:%M:%S')"
```

---

## Complete Security Scripts

### One-Click Enterprise Hardening Script

```bash
#!/bin/bash
# vps_enterprise_hardening.sh
# Run as root on a fresh VPS installation

set -e

echo "=========================================="
echo "  VPS Enterprise Security Hardening"
echo "  $(date '+%Y-%m-%d %H:%M:%S')"
echo "=========================================="

# --- Step 1: System Update ---
echo "[1/8] Updating system packages..."
export DEBIAN_FRONTEND=noninteractive
apt update && apt upgrade -y

# --- Step 2: Install Security Tools ---
echo "[2/8] Installing security tools..."
apt install -y ufw fail2ban curl wget vim git net-tools unzip     unattended-upgrades needrestart

# --- Step 3: SSH Hardening ---
echo "[3/8] SSH Hardening..."
read -p "Enter new SSH port (default: 22022): " SSH_PORT
SSH_PORT=${SSH_PORT:-22022}

# Backup original config
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Apply hardening settings
sed -i "s/^#Port 22/Port $SSH_PORT/" /etc/ssh/sshd_config
sed -i "s/^Port 22/Port $SSH_PORT/" /etc/ssh/sshd_config
sed -i "s/^#PermitRootLogin yes/PermitRootLogin no/" /etc/ssh/sshd_config
sed -i "s/^PermitRootLogin yes/PermitRootLogin no/" /etc/ssh/sshd_config
sed -i "s/^#PasswordAuthentication yes/PasswordAuthentication no/" /etc/ssh/sshd_config
sed -i "s/^PasswordAuthentication yes/PasswordAuthentication no/" /etc/ssh/sshd_config
sed -i "s/^X11Forwarding yes/X11Forwarding no/" /etc/ssh/sshd_config

# --- Step 4: Configure Firewall ---
echo "[4/8] Configuring firewall..."
ufw default deny incoming
ufw default allow outgoing
ufw allow $SSH_PORT/tcp comment 'SSH'
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'
ufw --force enable

# --- Step 5: Configure fail2ban ---
echo "[5/8] Configuring fail2ban..."
cat > /etc/fail2ban/jail.local <<'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
action = %(action_mwl)s

[sshd]
enabled = true
port = $SSH_PORT
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
EOF
systemctl restart fail2ban

# --- Step 6: Kernel Hardening ---
echo "[6/8] Applying kernel security optimizations..."
cat >> /etc/sysctl.conf <<'EOF'
# Security hardening
net.ipv4.tcp_syncookies = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
EOF
sysctl -p

# --- Step 7: Automated Security Updates ---
echo "[7/8] Configuring automatic security updates..."
dpkg-reconfigure -plow unattended-upgrades

# --- Step 8: Final Summary ---
echo "[8/8] Cleanup..."
apt autoremove -y
apt autoclean -y

echo ""
echo "=========================================="
echo "  ✅ Hardening Complete!"
echo "=========================================="
echo "  SSH Port: $SSH_PORT"
echo "  Firewall: ENABLED (deny incoming default)"
echo "  fail2ban: ENABLED"
echo ""
echo "  ⚠️  IMPORTANT:"
echo "  1. Keep your current SSH session open"
echo "  2. Test SSH login in a new terminal"
echo "  3. Install Google Authenticator for 2FA"
echo "  4. Copy your SSH public key to ~/.ssh/authorized_keys"
echo "  5. Add your IP to /etc/hosts.allow for SSH"
echo "=========================================="
```

---

## FAQ

### Q1: I'm locked out of my VPS after applying firewall rules. What do I do?

Don't panic — you have several recovery options:

| Solution | How | When to Use |
|----------|-----|-------------|
| **VNC Console** | Access via VPS control panel (SolusVM, Proxmox, etc.) | Always the first option |
| **Backup SSH Port** | Keep port 22 open as backup alongside your new port | Before enabling firewall |
| **Provider Rescue Mode** | Boot into recovery/rescue mode from control panel | When console is also blocked |
| **IP Whitelist** | Ensure your current IP is in hosts.allow | During initial setup |

Recovery via console:
```bash
# In VNC/console, disable UFW temporarily
ufw disable

# Or add your IP
ufw allow from YOUR_CURRENT_IP to any port 22022
```

### Q2: How do I unban myself from fail2ban?

```bash
# Check which jail banned you
fail2ban-client status

# Unban your IP from SSH jail
fail2ban-client set sshd unbanip YOUR_IP

# From all jails at once
fail2ban-client unban YOUR_IP

# View current bans
fail2ban-client banned

# Whitelist your IP to prevent future bans
echo "ignoreip = 127.0.0.1/8 1.2.3.4" >> /etc/fail2ban/jail.local
systemctl restart fail2ban
```

### Q3: How do I test my server's security?

| Test Type | Tool | Command |
|-----------|------|---------|
| Port Scanning | nmap | `nmap -sS -sV -O your-vps-ip` |
| SSH Audit | ssh-audit | `ssh-audit your-vps-ip` |
| SSL Rating | SSL Labs | Visit ssllabs.com/ssltest |
| DDoS Stress Test | — | ⚠️ Only test against your own server |
| Vulnerability Scan | lynis | `lynis audit system` |

### Q4: Should I use 2FA? What if I lose my phone?

Yes, 2FA is essential. To prevent being locked out:

- **Save backup codes** when setting up Google Authenticator
- **Use multiple authenticators**: Google Authenticator + Authy (which supports cloud backup)
- **Keep one SSH session open** when enabling 2FA until you've verified the new login works
- **Use a hardware key** (YubiKey) as a backup

### Q5: How often should I update security rules?

- **Weekly**: Review fail2ban logs and banned IPs
- **Monthly**: Update system packages and review firewall rules
- **After any incident**: Update rules to block new attack patterns
- **Quarterly**: Full security audit

---

## Recommended VPS Providers

| Provider | Strengths | Starting Price | Link |
|----------|-----------|---------------|------|
| **VPSVIP** | VPS reviews, benchmarks & buying guides | Varies | [Website](https://vpsvip.net) |
| **Vultr** | Hourly billing, global 25+ locations | $3.50/mo | [Website](https://vultr.com) |
| **DigitalOcean** | Developer-friendly, great documentation | $4/mo | [Website](https://digitalocean.com) |
| **Hetzner** | Excellent value, EU-based | €3.7/mo | [Website](https://hetzner.com) |
| **Linode** | Reliability, good performance | $5/mo | [Website](https://linode.com) |

---

## License

[CC BY-NC-SA 4.0](LICENSE) - Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International

You are free to:
- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:
- **Attribution** — You must give appropriate credit to this repository
- **NonCommercial** — You may not use the material for commercial purposes
- **ShareAlike** — If you remix or transform, you must distribute under the same license

---

<p align="center">
  <a href="https://vpsvip.net">VPSVIP</a> |
  <a href="https://clashvip.net">ClashVIP</a> |
  <a href="https://clashhub.net">ClashHub</a>
</p>

---

> For more VPS & Clash tools, check out [Awesome VPS & Clash Tools](https://github.com/CG-spring/awesome-vps-clash-tools)
