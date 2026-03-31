# VPS Security Pro - Advanced Guide

[![License](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/CG-spring/vps-security-pro.svg?style=flat-square)](https://github.com/CG-spring/vps-security-pro/stargazers)

> Enterprise-level security hardening, firewall configuration and advanced protection strategies
> 
> Continuously Updated | Last Updated: 2026-03-31

**Chinese** | **[English](README_EN.md)**

---

## Table of Contents

- [Advanced Security](#advanced-security)
- [SSH Hardening](#ssh-hardening)
- [Enterprise Firewall](#enterprise-firewall)
- [fail2ban Advanced](#fail2ban-advanced)
- [DDoS Protection](#ddos-protection)
- [Security Monitoring](#security-monitoring)
- [Scripts Collection](#scripts-collection)
- [FAQ](#faq)

---

## SSH Hardening

### Two-Factor Authentication

```bash
apt install -y libpam-google-authenticator
vim /etc/pam.d/sshd
# Add: auth required pam_google_authenticator.so
```

### IP Whitelist

```bash
# /etc/hosts.allow
sshd: 1.2.3.4 :allow

# /etc/hosts.deny
sshd: ALL :deny
```

---

## Enterprise Firewall

### UFW Advanced Rules

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 1.2.3.4 to any port 22
ufw limit from any to any port 22
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### iptables Script

```bash
#!/bin/bash
iptables -F
iptables -P INPUT DROP
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -m recent --set
iptables -A INPUT -p tcp --dport 22 -m recent --update --seconds 60 --hitcount 10 -j DROP
iptables-save > /etc/iptables/rules.v4
```

---

## fail2ban Advanced

### Multi-Jail Configuration

```bash
# /etc/fail2ban/jail.local

[DEFAULT]
bantime = 3600
maxretry = 3

[sshd]
enabled = true
port = 22
bantime = 86400

[nginx-http-auth]
enabled = true
port = http,https
```

---

## DDoS Protection

### Nginx Protection

```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

server {
    limit_conn addr 10;
    limit_req zone=one burst=20 nodelay;
    client_max_body_size 10M;
}
```

### Kernel Optimization

```bash
# /etc/sysctl.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_max_syn_backlog = 8192
sysctl -p
```

---

## Security Scripts

### One-Click Hardening

```bash
#!/bin/bash
echo "=== VPS Enterprise Security Hardening ==="

apt update && apt upgrade -y
apt install -y ufw fail2ban

read -p "Enter new SSH port: " SSH_PORT

sed -i "s/^#Port 22/Port $SSH_PORT/" /etc/ssh/sshd_config
sed -i "s/^#PermitRootLogin yes/PermitRootLogin no/" /etc/ssh/sshd_config

ufw default deny incoming
ufw allow $SSH_PORT/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable

cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
bantime = 3600
maxretry = 3

[sshd]
enabled = true
port = $SSH_PORT
EOF

systemctl restart fail2ban
echo "=== Hardening Complete ==="
```

---

## FAQ

### Q1: Locked out?

| Solution | Description |
|----------|-------------|
| VNC Console | Emergency login via VPS panel |
| Backup Port | Keep port 22 as backup |

### Q2: False ban?

```bash
fail2ban-client set sshd unbanip YOUR_IP
```

---

## Recommended VPS

| Name | Features | Price | Link |
|------|----------|-------|------|
| **VPSVIP** | VPS Reviews - Server benchmark and buying guide | [Website](https://vpsvip.net) |
| **Vultr** | Hourly billing | $3.5/mo | [Website](https://vultr.com) |

---

## License

[CC BY-NC-SA 4.0](LICENSE) - 2026

<p align="center">
  <a href="https://vpsvip.net">VPSVIP</a> |
  <a href="https://clashvip.net">ClashVIP</a>
</p>
