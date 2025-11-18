# 🛡️ **Full Server Hardening Guide for Production**

### *For: Next.js + FastAPI + Traefik + Docker (Rootful) + Ubuntu Server*

---

## ⚠️ Disclaimer

This guide assumes:

* OS: **Ubuntu 22.04+**
* Deployment: **Docker rootful**
* Reverse proxy: **Traefik**
* Apps: **Next.js, FastAPI, PostgreSQL**
* No Cloudflare (direct server exposure)

---

# 🧾 Table of Contents

1. [Update & System Basics](#1-update--system-basics)
2. [Create a Non-Root Admin User](#2-create-a-nonroot-admin-user)
3. [SSH Hardening](#3-ssh-hardening)
4. [UFW Firewall Setup](#4-ufw-firewall-setup)
5. [Docker Firewall Fix (DOCKER-USER Chain)](#5-docker-firewall-fix-dockeruser-chain)
6. [Kernel Hardening (sysctl)](#6-kernel-hardening-sysctl)
7. [Fail2Ban Setup](#7-fail2ban-setup)
8. [Traefik Security Hardening](#8-traefik-security-hardening)
9. [Docker Daemon Hardening](#9-docker-daemon-hardening)
10. [Secure PostgreSQL](#10-secure-postgresql)
11. [Log Monitoring & Alerts](#11-log-monitoring--alerts)
12. [Backups & Disaster Recovery](#12-backups--disaster-recovery)

---

# 1. **Update & System Basics**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

# 2. **Create a Non-Root Admin User**

```bash
adduser admin
usermod -aG sudo admin
```

Disable root login later in SSH.

---

# 3. **SSH Hardening**

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Recommended settings:

```
Port 22
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
AllowUsers admin
MaxAuthTries 3
LoginGraceTime 20
```

Apply:

```bash
sudo systemctl restart ssh
```

---

# 4. **UFW Firewall Setup**

Default policy:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow only required ports:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Enable:

```bash
sudo ufw enable
```

Verify:

```bash
sudo ufw status verbose
```

---

# 5. **Docker Firewall Fix (DOCKER-USER Chain)**

**Critical: Docker bypasses UFW by default.**

Block unwanted container ports fully:

```bash
sudo iptables -I DOCKER-USER -i eth0 -p tcp --dport 5432 -j DROP
sudo iptables -I DOCKER-USER -i eth0 -p tcp --dport 8000 -j DROP
sudo iptables -I DOCKER-USER -i eth0 -p tcp --dport 3000 -j DROP
```

Allow container-to-container traffic:

```bash
sudo iptables -A DOCKER-USER -i docker0 -j ACCEPT
```

Persist rules:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

# 6. **Kernel Hardening (sysctl)**

Create:

```bash
sudo nano /etc/sysctl.d/99-custom.conf
```

Add:

```
# Disable ping
net.ipv4.icmp_echo_ignore_all = 1

# Mitigate SYN flood
net.ipv4.tcp_syncookies = 1

# Disable IP source routing
net.ipv4.conf.all.accept_source_route = 0

# Disable redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Enable ASLR
kernel.randomize_va_space = 2

# Disable IPv6 (optional)
net.ipv6.conf.all.disable_ipv6 = 1
```

Apply:

```bash
sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

---

# 7. **Fail2Ban Setup**

Install:

```bash
sudo apt install fail2ban -y
```

Create:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add:

```
[sshd]
enabled = true
port = 22
filter = sshd
maxretry = 4
bantime = 1h
findtime = 10m
```

Enable:

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

---

# 8. **Traefik Security Hardening**

### 8.1 Add secure headers middleware

File: `traefik/dynamic/middlewares.yml`

```yaml
http:
  middlewares:
    securedHeaders:
      headers:
        frameDeny: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "no-referrer"
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
```

Use on all routers:

```yaml
traefik.http.routers.web.middlewares=securedHeaders@file
```

---

### 8.2 Enable Traefik Dashboard protection

Password:

```bash
echo $(htpasswd -nb admin strongpassword)
```

Add to dashboard router:

```yaml
traefik.http.routers.traefik.middlewares=auth@file
```

Middleware:

```yaml
http:
  middlewares:
    auth:
      basicAuth:
        users:
          - "admin:HASHED_PASSWORD_HERE"
```

---

# 9. **Docker Daemon Hardening**

File:

```
sudo nano /etc/docker/daemon.json
```

Recommended:

```json
{
  "icc": false,
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

---

# 10. **Secure PostgreSQL**

### Do NOT expose Postgres:

In docker-compose:

```yaml
db:
  ports: []  # remove "5432:5432"
```

### Allow only internal Docker access:

```
host all all 172.18.0.0/16 md5
```

(Replace with your docker network CIDR)

---

# 11. **Log Monitoring & Alerts**

Install:

```bash
sudo apt install logwatch -y
```

Edit:

```bash
sudo nano /etc/cron.daily/00logwatch
```

Use mail notifications.

Optional SIEM-like:

* Vector.dev
* Loki + Promtail
* Datadog
* Better uptime alerting for crashes

---

# 12. **Backups & Disaster Recovery**

### Database dump

Daily backup:

```bash
sudo -u postgres pg_dumpall > /backups/db-$(date +\%F).sql
```

### Backup Docker volumes

```
docker run --rm -v pgdata:/volume -v $(pwd):/backup ubuntu bash -c \
  "tar cvf /backup/pgdata.tar /volume"
```

### Offsite backup:

* Backblaze B2
* AWS S3
* Wasabi

Automatically with restic:

```
restic -r b2:bucket backup /backups
```

