# VPS 安全防护进阶指南

[![License](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/CG-spring/vps-security-pro.svg?style=flat-square)](https://github.com/CG-spring/vps-security-pro/stargazers)

> VPS 安全防护进阶指南 - 企业级安全加固、防火墙配置与高级防护策?> 
> 持续更新?| 最后更? 2026-03-31

**中文** | **[English](README_EN.md)**

---

## 目录

- [为什么需要进阶安全防护？](#为什么需要进阶安全防?
- [SSH 高级加固](#ssh-高级加固)
- [企业级防火墙配置](#企业级防火墙配置)
- [fail2ban 高级配置](#fail2ban-高级配置)
- [DDoS 防护方案](#ddos-防护方案)
- [安全监控与告警](#安全监控与告?
- [安全脚本集合](#安全脚本集合)
- [常见问题](#常见问题)

---

## 为什么需要进阶安全防护？

### 普通防?vs 企业级防?
| 防护层级 | 普通方?| 企业级方?|
|----------|----------|------------|
| SSH 加固 | 修改端口 | 密钥+IP白名?|
| 防火?| 基础端口 | 复杂规则+限流 |
| 入侵检?| ?| 行为分析+告警 |
| 日志分析 | ?| 集中日志+分析 |
| 备份 | 手动 | 自动+异地 |

### 常见攻击类型

| 攻击类型 | 危害 | 防御方案 |
|----------|------|----------|
| SSH 暴力破解 | 盗取服务?| 密钥+fail2ban |
| DDoS 攻击 | 服务不可?| 流量清洗+限?|
| 供应链攻?| 植入后门 | 安全更新 |
| 0day 漏洞 | 未知风险 | 最小化暴露?|

---

## SSH 高级加固

### 1. 双因素认?(2FA)

```bash
# 安装 Google Authenticator
apt install -y libpam-google-authenticator

# 配置 PAM
vim /etc/pam.d/sshd

# 添加以下?auth required pam_google_authenticator.so

# 配置 SSH
vim /etc/ssh/sshd_config

# 修改以下配置
ChallengeResponseAuthentication yes
AuthenticationMethods password,keyboard-interactive

# 重启 SSH
systemctl restart sshd
```

### 2. SSH 密钥 + 密钥短语

```bash
# 生成带密钥短语的密钥?ssh-keygen -t ed25519 -f ~/.ssh/vps_master -C "vps-master-key"

# 密钥短语：使用密码管理器保存

# 上传公钥
ssh-copy-id -i ~/.ssh/vps_master.pub admin@your-vps-ip

# 配置 SSH 使用特定密钥
vim ~/.ssh/config

Host your-vps-ip
    IdentityFile ~/.ssh/vps_master
```

### 3. IP 白名?+ 地理封锁

```bash
# 只允许特?IP 访问 SSH
vim /etc/hosts.allow

sshd: 1.2.3.4 :allow
sshd: 10.0.0.0/8 :allow

vim /etc/hosts.deny

sshd: ALL :deny

# 或者使?UFW 地理封锁
ufw deny from 某些国家 to any port 22
```

---

## 企业级防火墙配置

### 1. UFW 高级规则

```bash
# 基础配置
ufw default deny incoming
ufw default allow outgoing

# 开放指?IP ?SSH (推荐)
ufw allow from 1.2.3.4 to any port 22 proto tcp

# 限流规则 - 防止暴力破解
ufw limit from any to any port 22 proto tcp

# 开?HTTP/HTTPS
ufw allow 80/tcp
ufw allow 443/tcp

# 开放常用服务端?ufw allow 3306/tcp comment 'MySQL'
ufw allow 5432/tcp comment 'PostgreSQL'
ufw allow 6379/tcp comment 'Redis'
ufw allow 27017/tcp comment 'MongoDB'

# 启用防火?ufw enable
```

### 2. iptables 高级规则

```bash
# 基础防护脚本
#!/bin/bash

# 清空现有规则
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# 默认策略
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH 限流 (每分?10 ?
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 防止 SYN 洪水攻击
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# 防止 Ping 洪水
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# 开放服务端?iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 保存规则
iptables-save > /etc/iptables/rules.v4
```

### 3. 自动防御脚本

```bash
#!/bin/bash
# 自动封锁攻击?IP

LOG_FILE="/var/log/auth.log"
BLOCKED_IPS="/tmp/blocked_ips.txt"

# 获取过去 1 小时尝试登录失败?IP
BLOCKED=$(grep "Failed password" $LOG_FILE | grep "$(date -d '1 hour ago' +'%b %d')" | awk '{print $11}' | sort | uniq -c | awk '$1>10 {print $2}')

# 封锁 IP
for IP in $BLOCKED; do
    if ! grep -q $IP $BLOCKED_IPS; then
        echo $IP >> $BLOCKED_IPS
        ufw insert 1 deny from $IP to any comment "Auto-blocked"
        echo "$(date): Blocked $IP (failed login attempts)"
    fi
done
```

---

## fail2ban 高级配置

### 1. 多监狱配?
```bash
# /etc/fail2ban/jail.local

[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
destemail = admin@your-domain.com
sender = fail2ban@your-domain.com
action = %(action_mwl)s

# SSH 监狱
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400

# HTTP 暴力破解
[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 5

# WordPress 保护
[wp-login]
enabled = true
port = http,https
filter = wp-login
logpath = /var/log/nginx/access.log
maxretry = 5
bantime = 7200

# vsFTPd 保护
[vsftpd]
enabled = true
port = ftp,ftp-data,21
filter = vsftpd
logpath = /var/log/vsftpd.log
maxretry = 5
```

### 2. 自定义过滤器

```bash
# /etc/fail2ban/filter.d/wp-login.conf

[Definition]
failregex = ^<HOST> - - \[.*\] "POST /wp-login.php
ignoreregex =
```

---

## DDoS 防护方案

### 1. 应用层防?
```nginx
# Nginx DDoS 防护配置

# 限制连接?limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

server {
    # 连接数限?    limit_conn addr 10;
    
    # 请求频率限制
    limit_req zone=one burst=20 nodelay;
    
    # 请求体大小限?    client_max_body_size 10M;
    
    # 超时设置
    client_body_timeout 10s;
    client_header_timeout 10s;
    
    # 防盗?    valid_referers none blocked your-domain.com;
    if ($invalid_referer) {
        return 403;
    }
}
```

### 2. 系统级防?
```bash
# /etc/sysctl.conf DDoS 防护

# 防止 SYN 洪水
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_synack_retries = 2

# 限制连接?net.ipv4.ip_local_port_range = 2000 65000
net.ipv4.tcp_max_tw_buckets = 2000
net.ipv4.tcp_max_syn_backlog = 8192

# ICMP 限制
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# 应用配置
sysctl -p
```

---

## 安全监控与告?
### 1. 实时安全仪表?
```bash
#!/bin/bash
# VPS 安全状态仪表板

echo "=========================================="
echo "     VPS 安全监控仪表?
echo "     $(date '+%Y-%m-%d %H:%M:%S')"
echo "=========================================="
echo ""

# SSH 连接状?echo "[SSH 安全状态]"
echo "-----------------------------"
echo "当前 SSH 连接? $(who | wc -l)"
echo "SSH 失败登录 (今天): $(grep -c "$(date +%b\ %d)" /var/log/auth.log | head -1)"
echo "fail2ban 封禁 IP ? $(fail2ban-client banned | wc -l)"
echo ""

# 网络连接
echo "[网络连接状态]"
echo "-----------------------------"
echo "TCP 连接? $(netstat -an | grep tcp | wc -l)"
echo "UDP 连接? $(netstat -an | grep udp | wc -l)"
echo "ESTABLISHED: $(netstat -an | grep ESTABLISHED | wc -l)"
echo ""

# 端口扫描检?echo "[可疑端口]"
echo "-----------------------------"
netstat -tuln | awk '{print $1,$4,$6}' | grep LISTEN
echo ""

# 防火墙状?echo "[防火墙状态]"
echo "-----------------------------"
ufw status | head -10
echo ""

# 最近安全事?echo "[最近安全事件]"
echo "-----------------------------"
tail -5 /var/log/auth.log | grep -i "failed\|error\|attack"
echo ""

echo "=========================================="
```

### 2. Telegram 告警机器?
```bash
#!/bin/bash
# 安全告警通知脚本

TELEGRAM_BOT_TOKEN="YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID="YOUR_CHAT_ID"

send_alert() {
    MESSAGE="$1"
    curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
        -d "chat_id=$TELEGRAM_CHAT_ID&text=$MESSAGE&parse_mode=HTML"
}

# 使用示例
send_alert "⚠️ <b>VPS 安全告警</b>%0A检测到 SSH 暴力破解攻击%0AIP: 192.168.1.100%0A尝试次数: 50"
```

---

## 安全脚本集合

### 完整安全加固脚本

```bash
#!/bin/bash
# VPS 企业级安全加固脚?
echo "=========================================="
echo "  VPS 企业级安全加固脚?
echo "=========================================="

# 1. 更新系统
echo "[1/8] 更新系统..."
apt update && apt upgrade -y

# 2. 安装安全工具
echo "[2/8] 安装安全工具..."
apt install -y ufw fail2ban curl wget vim git net-tools

# 3. SSH 加固
echo "[3/8] SSH 加固..."
read -p "请输入新 SSH 端口: " SSH_PORT

sed -i "s/^#Port 22/Port $SSH_PORT/" /etc/ssh/sshd_config
sed -i "s/^Port 22/Port $SSH_PORT/" /etc/ssh/sshd_config
sed -i "s/^#PermitRootLogin yes/PermitRootLogin no/" /etc/ssh/sshd_config
sed -i "s/^#PasswordAuthentication yes/PasswordAuthentication no/" /etc/ssh/sshd_config

# 4. 配置防火?echo "[4/8] 配置防火?.."
ufw default deny incoming
ufw default allow outgoing
ufw allow $SSH_PORT/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable

# 5. 配置 fail2ban
echo "[5/8] 配置 fail2ban..."
cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = $SSH_PORT
bantime = 86400
EOF
systemctl restart fail2ban

# 6. 内核优化
echo "[6/8] 内核安全优化..."
cat >> /etc/sysctl.conf <<EOF
net.ipv4.tcp_syncookies = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.tcp_max_syn_backlog = 8192
EOF
sysctl -p

# 7. 安装 Fail2ban Web 界面 (可?
# echo "[7/8] 安装 Fail2ban Web..."
# apt install -y fail2ban-ui

# 8. 设置自动更新
echo "[8/8] 配置自动安全更新..."
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades

echo ""
echo "=========================================="
echo "  安全加固完成!"
echo "=========================================="
echo "SSH 端口: $SSH_PORT"
echo "请保存好 SSH 密钥!"
echo "请重启服务器使所有配置生?"
```

---

## 常见问题

### Q1: 自己无法登录服务器怎么办？

| 解决方案 | 说明 |
|----------|------|
| 使用 VNC 控制?| VPS 控制台提供的紧急登?|
| 保留备用 SSH 端口 | 22 端口备用 |
| IP 白名?| 确保当前 IP 在白名单 |

### Q2: 误封自己怎么办？

```bash
# 解封 IP
fail2ban-client set sshd unbanip YOUR_IP

# 查看被封 IP
fail2ban-client banned

# 临时关闭 fail2ban
fail2ban-client stop
```

### Q3: 如何测试服务器安全性？

| 测试项目 | 工具/网站 |
|----------|----------|
| 端口扫描 | nmap -sS -sV -O your-vps-ip |
| SSL 评级 | ssllabs.com/ssltest/ |
| SSH 漏洞 | ssh-audit your-vps-ip |
| DDoS 压力测试 | 请谨慎使?|

---

## 推荐 VPS

| 名称 | 特点 | 价格 | 链接 |
|------|------|------|------|
| **VPSVIP** | VPS主机测评 | 服务器评测与推荐 | [官网](https://vpsvip.net) |
| **Vultr** | 按小时计费，全球节点 | $3.5/月起 | [官网](https://vultr.com) |
| **BandwagonHost** | 性价比高 | $49.99/?| [官网](https://bandwagonhost.com) |

---

## License

[CC BY-NC-SA 4.0](LICENSE) - 2026

---

<p align="center">
  <a href="https://vpsvip.net">VPSVIP</a> |
  <a href="https://clashvip.net">ClashVIP</a> |
  <a href="https://clashhub.net">ClashHub</a>
</p>
