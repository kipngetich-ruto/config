# 🧑‍💻 **MASTER — Full Development → Deployment Pipeline**

This documents the **entire setup** for:

* Local development environment (Lubuntu + Rootless Docker)
* Building production containers
* Deploying to a VPS with Docker + Compose
* Reverse proxy + HTTPS using Traefik
* Managing multiple GitHub accounts
* CI/CD pipeline for automated deployments

Designed for:

* **Node.js**, **Python/FastAPI**, **PostgreSQL**, **Redis**, **Next.js**, **Nuxt**, etc.
* Full-stack SaaS / personal projects
* Secure, scalable production deployments

---

# 🐧 **1. Rootless Docker Setup on Lubuntu (Local Development)**

Rootless Docker is ideal for laptops and dual-boot machines because it avoids root privileges while giving full development functionality.

⚠ **Not for production.** Use rootful Docker on VPS.

---

## ✓ 1.1 Install Dependencies

```bash
sudo apt update
sudo apt install -y uidmap dbus-user-session slirp4netns fuse-overlayfs iptables
```

---

## ✓ 1.2 Install Rootless Docker

```bash
curl -fsSL https://get.docker.com/rootless | sh
```

---

## ✓ 1.3 Enable Rootless Docker Service

```bash
systemctl --user enable docker
systemctl --user start docker
sudo loginctl enable-linger $USER
```

---

## ✓ 1.4 Add Environment Variables

Add to `~/.bashrc`:

```bash
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix:///run/user/$UID/docker.sock
```

Reload:

```bash
source ~/.bashrc
```

✔ This is the recommended method  
✔ Ignore the "context override" warning  
✔ DOCKER_HOST is correct for rootless mode

---

## ✓ 1.5 Install Docker Compose v2

```bash
mkdir -p ~/.docker/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 \
  -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
```

Check:

```bash
docker compose version
```

---

## ✓ 1.6 Test rootless Docker

```bash
docker run hello-world
```

Must show "Hello from Docker!".

---

## ⚠ Rootless Limitations (Expected)

| Feature              | Supported   |
| -------------------- | ----------- |
| Ports > 1024         | ✔ Yes       |
| Ports 80/443         | ❌ No        |
| Traefik / Nginx      | ❌ Not ideal |
| Production workloads | ❌ No        |
| Local dev            | ✔ Perfect   |

---

# 🔐 **2. GitHub SSH Key Setup (Multi-Account + Signing)**

Supports:

* Personal GitHub
* Work GitHub
* SSH authentication
* SSH commit signing

---

## ✓ 2.1 Create two SSH keys

Personal:

```bash
ssh-keygen -t ed25519 -C "personal@email.com" -f ~/.ssh/id_ed25519_personal
```

Work:

```bash
ssh-keygen -t ed25519 -C "work@email.com" -f ~/.ssh/id_ed25519_work
```

---

## ✓ 2.2 Add keys to agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

---

## ✓ 2.3 SSH Config

`nano ~/.ssh/config`

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

Clone example:

```bash
git clone git@github.com-personal:username/repo.git
git clone git@github.com-work:company/repo.git
```

---

## ✓ 2.4 SSH Signing per repo

Personal repo:

```bash
git config user.signingkey ~/.ssh/id_ed25519_personal.pub
```

Work repo:

```bash
git config user.signingkey ~/.ssh/id_ed25519_work.pub
```

Global signing:

```bash
git config --global gpg.format ssh
git config --global commit.gpgsign true
```

---

# 🏭 **3. VPS Deployment Guide (Production)**

Production uses **rootful Docker** (required for port 80/443, SSL, reverse proxy).

Works on Ubuntu 22.04/24.04 servers.

---

## ✓ 3.1 Install Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

Add Docker repo:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
```

Add user to group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## ✓ 3.2 Install Docker Compose (Standalone - Optional)

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.40.3/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Note: Modern Docker installations include `docker compose` (plugin) by default via `docker-compose-plugin` package.

---

## ✓ 3.3 Install Traefik (Automatic HTTPS)

```bash
mkdir -p ~/apps/traefik
cd ~/apps/traefik
```

`docker-compose.yml`:

```yaml
services:
  traefik:
    image: traefik:v2.11
    restart: always
    container_name: traefik
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.leresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.leresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.leresolver.acme.email=YOUR_EMAIL@example.com"
      - "--certificatesresolvers.leresolver.acme.storage=/acme.json"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
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

Create network and start:

```bash
docker network create web
touch acme.json
chmod 600 acme.json
docker compose up -d
```

---

## ✓ 3.4 Deploy your app (example)

`docker-compose.yml`:

```yaml
services:
  myapp:
    image: username/myapp:latest
    restart: always
    container_name: myapp
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`domain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=leresolver"
      - "traefik.http.services.myapp.loadbalancer.server.port=3000"
    networks:
      - web

networks:
  web:
    external: true
```

Deploy:

```bash
docker compose up -d
```

---

# 📦 **4. Docker Compose Templates (Local & Production)**

## ✓ Node.js

```yaml
services:
  web:
    image: node:24-alpine
    working_dir: /app
    volumes:
      - .:/app
      - /app/node_modules
    command: sh -c "npm install && npm run dev"
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
```

## ✓ FastAPI

```yaml
services:
  api:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - .:/app
    command: sh -c "pip install -r requirements.txt && uvicorn main:app --host 0.0.0.0 --port 8000 --reload"
    ports:
      - "8000:8000"
    environment:
      - PYTHONUNBUFFERED=1
```

## ✓ PostgreSQL

```yaml
services:
  postgres:
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: db
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## ✓ Redis

```yaml
services:
  redis:
    image: redis:8.2-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - ./redis-data:/data
```

---

# 🚀 **5. CI/CD: GitHub Actions → VPS**

## ✓ 5.1 Create SSH key for CI/CD

```bash
ssh-keygen -t ed25519 -f deploy_key -N ""
```

Add public key to VPS:

```bash
cat deploy_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Add secrets in GitHub (Settings → Secrets → Actions):

```
VPS_HOST          (e.g., 1.2.3.4 or domain.com)
VPS_USER          (e.g., ubuntu or your-username)
VPS_SSH_KEY       (contents of deploy_key - private key)
DOCKER_USER       (Docker Hub username)
DOCKER_PASS       (Docker Hub password or access token)
```

---

## ✓ 5.2 GitHub Actions Workflow

`.github/workflows/deploy.yml`

```yaml
name: Deploy to VPS

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USER }}/myapp:latest,${{ secrets.DOCKER_USER }}/myapp:${{ github.sha }}
          cache-from: type=registry,ref=${{ secrets.DOCKER_USER }}/myapp:buildcache
          cache-to: type=registry,ref=${{ secrets.DOCKER_USER }}/myapp:buildcache,mode=max

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd ~/apps/myapp
            docker compose pull
            docker compose up -d --force-recreate
            docker image prune -f
```

---

## ✓ 5.3 Example Dockerfile

```dockerfile
FROM node:24-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

FROM node:24-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production

COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

---

# 🎉 **Your Entire Workflow is Now Complete**

You now have:

### ✔ Local dev environment (Lubuntu + Rootless Docker)

### ✔ Secure GitHub SSH + signing setup

### ✔ Docker Compose templates for any stack

### ✔ VPS production deployment with Docker + Traefik + HTTPS

### ✔ GitHub Actions CI/CD auto-deploy

---

## 📝 **Quick Reference Commands**

### Local Development
```bash
# Start services
docker compose up -d

# View logs
docker compose logs -f

# Stop services
docker compose down
```

### Production Deployment
```bash
# Pull latest image
docker compose pull

# Recreate containers
docker compose up -d --force-recreate

# Check status
docker compose ps

# View logs
docker compose logs -f
```

### Maintenance
```bash
# Remove unused images
docker image prune -a -f

# Remove unused volumes
docker volume prune -f

# System cleanup
docker system prune -a -f --volumes
```