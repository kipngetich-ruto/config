# 🧑‍💻 **MASTER — Full Development → Deployment Pipeline (2025 Edition)**

This document covers:

* Local dev environment (Lubuntu + Rootless Docker)
* GitHub SSH + multi-account + commit signing
* Docker Compose templates for Node, Python, PostgreSQL, Redis
* Deployment to VPS with Docker + Traefik v3.5.6
* Production `.env` templates
* Automatic backups (Postgres 18.1 + Redis 8.2.3)
* Zero-downtime deployment (Blue–Green)
* CI/CD auto-deploy from GitHub Actions

---

# 🐧 **1. Local Development — Lubuntu + Rootless Docker**

Rootless Docker is recommended for laptops because it:

* runs without `sudo`
* avoids overwriting system files
* isolates user-space containers
* is perfect for local dev

⚠ Rootless Docker **cannot bind to ports 80/443**.
For production, use rootful Docker.

---

## ✓ 1.1 Install Dependencies

```bash
sudo apt update
sudo apt install -y uidmap dbus-user-session slirp4netns fuse-overlayfs iptables
```

## ✓ 1.2 Install Rootless Docker

```bash
curl -fsSL https://get.docker.com/rootless | sh
```

## ✓ 1.3 Enable Docker Service

```bash
systemctl --user enable docker
systemctl --user start docker
sudo loginctl enable-linger $USER
```

## ✓ 1.4 Add to ~/.bashrc

```bash
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix:///run/user/$UID/docker.sock
```

Reload:

```bash
source ~/.bashrc
```

## ✓ 1.5 Install Docker Compose v2

```bash
mkdir -p ~/.docker/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.40.3/docker-compose-linux-x86_64 \
  -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
```

---

# 🔐 **2. GitHub SSH + Multi-Account + Commit Signing**

## ✓ 2.1 Create Keys

```bash
ssh-keygen -t ed25519 -C "personal_email" -f ~/.ssh/id_ed25519_personal
ssh-keygen -t ed25519 -C "work_email" -f ~/.ssh/id_ed25519_work
```

## ✓ 2.2 SSH Config

`~/.ssh/config`

```ssh
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes

Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

## ✓ 2.3 Commit Signing

Global:

```bash
git config --global gpg.format ssh
git config --global commit.gpgsign true
```

Per repo:

```bash
git config user.signingkey ~/.ssh/id_ed25519_personal.pub
```

---

# 🏭 **3. VPS Deployment (Production) — Docker + Traefik v3.5.6**

Works on Ubuntu 22.04 / 24.04 / 24.10.

---

## ✓ 3.1 Install Docker (rootful)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

Add Docker repo:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
 | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Add repo:

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
 | sudo tee /etc/apt/sources.list.d/docker.list
```

Install Docker:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Enable:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add your user:

```bash
sudo usermod -aG docker $USER
```

---

# 🌐 **4. Traefik v3.5.6 — Latest Stable Reverse Proxy (HTTPS)**

This configuration gives:

✔ Automatic HTTPS
✔ HTTP → HTTPS redirect
✔ Dashboard
✔ Easy multi-app routing
✔ Supports Node, Python, Go, PHP, Rust, anything

---

## 📂 Folder structure

```
/home/youruser/traefik/
│── docker-compose.yml
│── acme.json
```

---

## 🚀 **Traefik v3.5.6 — docker-compose.yml (2025)**

```yaml
services:
  traefik:
    image: traefik:v3.5.6
    container_name: traefik
    restart: always
    command:
      - "--api.dashboard=true"

      # Providers
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"

      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

      # Redirect HTTP → HTTPS
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"

      # SSL (Let’s Encrypt)
      - "--certificatesresolvers.leresolver.acme.email=${TRAEFIK_EMAIL}"
      - "--certificatesresolvers.leresolver.acme.storage=/acme.json"
      - "--certificatesresolvers.leresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.leresolver.acme.httpchallenge.entrypoint=web"

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./acme.json:/acme.json
      - /var/run/docker.sock:/var/run/docker.sock:ro

    networks:
      - web

networks:
  web:
    external: true
```

---

# 🏗 **5. App Deployment (Node / FastAPI / Anything)**

Example app exposed behind Traefik:

```yaml
services:
  myapp:
    image: username/myapp:latest
    restart: always
    environment:
      DATABASE_URL: postgres://pguser:pgpass@postgres:5432/appdb
      REDIS_URL: redis://redis:6379
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`mydomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=leresolver"
      - "traefik.http.services.myapp.loadbalancer.server.port=3000"
    networks:
      - web
```

---

# 📦 **6. Database Services**

## PostgreSQL 18.1 — Latest LTS (2025)

```yaml
services:
  postgres:
    image: postgres:18.1
    restart: unless-stopped
    environment:
      POSTGRES_USER: pguser
      POSTGRES_PASSWORD: pgpass
      POSTGRES_DB: appdb
    volumes:
      - ./pgdata:/var/lib/postgresql/data
```

## Redis 8.2.3 — Latest Stable

```yaml
services:
  redis:
    image: redis:8.2.3-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - ./redis-data:/data
```

---

# 🗄️ **7. Automatic Backups**

## ✓ PostgreSQL backup script

`~/postgres/backup.sh`

```bash
#!/bin/bash
BACKUP_DIR="/home/youruser/postgres/backups"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")

docker exec postgres pg_dump -U pguser -d appdb \
 > "$BACKUP_DIR/backup_$TIMESTAMP.sql"

find $BACKUP_DIR -type f -mtime +14 -delete
```

Cron:

```
0 2 * * * /home/youruser/postgres/backup.sh
```

## ✓ Redis backup script

`~/redis/backup.sh`

```bash
#!/bin/bash
BACKUP_DIR="/home/youruser/redis/backups"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")

docker exec redis redis-cli save
docker cp redis:/data/dump.rdb \
 "$BACKUP_DIR/dump_$TIMESTAMP.rdb"

find $BACKUP_DIR -type f -mtime +10 -delete
```

Cron:

```
0 3 * * * /home/youruser/redis/backup.sh
```

---

# 🔄 **8. Zero-Downtime Deployment — Blue/Green Deploys**

Folder layout:

```
myapp/
├── blue/
├── green/
└── active -> blue
```

Deployment script:

```bash
#!/bin/bash
APP_DIR="/home/youruser/apps/myapp"
ACTIVE=$(readlink $APP_DIR/active)

if [[ "$ACTIVE" == "$APP_DIR/blue" ]]; then
    NEXT="green"
else
    NEXT="blue"
fi

docker compose -f "$APP_DIR/$NEXT/docker-compose.yml" pull
docker compose -f "$APP_DIR/$NEXT/docker-compose.yml" up -d --remove-orphans

ln -sfn "$APP_DIR/$NEXT" "$APP_DIR/active"
echo "Switched to $NEXT"
```

---

# 🚀 **9. CI/CD — GitHub → VPS**

`.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: ["main"]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USER }}
        password: ${{ secrets.DOCKER_PASS }}

    - uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKER_USER }}/myapp:latest

    - uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.VPS_HOST }}
        username: ${{ secrets.VPS_USER }}
        key: ${{ secrets.VPS_SSH_KEY }}
        script: |
          cd ~/apps/myapp
          ./deploy.sh
```
