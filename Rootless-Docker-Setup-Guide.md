# 🐳 **Rootless Docker Setup Guide (Lubuntu 24.04+)**

This document provides a secure and developer-friendly Docker setup for Lubuntu/Ubuntu systems using **Rootless Docker**, perfect for local development and building container images for deployment to a VPS.

Rootless Docker runs the Docker daemon **without root**, giving you much stronger isolation and system safety—great for dual-boot machines and laptops.

---

# 📌 **Why Use Rootless Docker?**

Rootless Docker is excellent for LOCAL development because:

* No `sudo` needed
* No system-wide privileges
* Much safer than rootful Docker
* Containers cannot damage the host
* Perfect isolation on a development laptop
* Ideal for Node.js, Python, FastAPI, PostgreSQL, Redis, and local microservices

However:

❌ **Do NOT use rootless Docker on a production VPS**
Because it cannot bind ports `< 1024` (80, 443, 53, etc.) and has limited kernel features.

---

# 🚀 **1. Install Dependencies**

```bash
sudo apt update
sudo apt install -y \
    uidmap \
    dbus-user-session \
    slirp4netns \
    fuse-overlayfs \
    iptables
```

---

# 🚀 **2. Install Rootless Docker**

```bash
curl -fsSL https://get.docker.com/rootless | sh
```

This installs the rootless daemon, rootlesskit, slirp4netns, and user-level systemd service files.

---

# 🚀 **3. Enable Docker Rootless Service**

```bash
systemctl --user enable docker
systemctl --user start docker
```

Enable lingering so Docker starts automatically:

```bash
sudo loginctl enable-linger $USER
```

---

# 🚀 **4. Export Required Environment Variables**

Add the following to your `~/.bashrc`:

```bash
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix:///run/user/$UID/docker.sock
```

Reload shell:

```bash
source ~/.bashrc
```

### ✔ This is the recommended method.

Docker will automatically talk to the rootless daemon through `DOCKER_HOST`.

> ⚠ Ignore the context-related warning:
>
> *“DOCKER_HOST overrides active context”*
>
> This is normal and **safe**. DOCKER_HOST is the correct way to use rootless Docker.

---

# 🔍 **5. Verify Installation**

```bash
docker info
docker run hello-world
```

Expected: “Hello from Docker!” message.

---

# 🔧 **6. Install Docker Compose v2**

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

# 🔥 **7. Running Containers in Rootless Mode**

Rootless Docker **only binds ports > 1024**.

Examples:

✔ Works:

```bash
docker run -p 3000:3000 node:alpine
docker run -p 8080:8080 nginx
```

❌ Doesn’t work:

```bash
docker run -p 80:80 nginx     # requires root
```

You can still run a reverse proxy that listens on 3000 → 80 later on a VPS.

---

# 🧠 **8. DOCKER_HOST vs Context (Important)**

Rootless Docker can be used in two modes:

### **A) Using DOCKER_HOST (Recommended)**

This is the default and most stable approach:

```bash
export DOCKER_HOST=unix:///run/user/$UID/docker.sock
```

### **B) Using Docker Contexts**

You can switch context:

```bash
docker context use rootless
```

But this will produce a warning if `DOCKER_HOST` is set.

### ✔ Best practice:

👉 **Use DOCKER_HOST and ignore the warning**
It is expected and harmless.

---

# 🛠 **9. Troubleshooting**

### **AppArmor blocking rootlesskit**

If you see:

```
apparmor_restrict_unprivileged_userns=1
permission denied
rootlesskit failed
```

Fix by creating an AppArmor exception:

```bash
cat <<EOT | sudo tee "/etc/apparmor.d/home.$USER.bin.rootlesskit"
abi <abi/4.0>,
include <tunables/global>

/home/$USER/bin/rootlesskit flags=(unconfined) {
  userns,
  include if exists <local/home.$USER.bin.rootlesskit>
}
EOT

sudo systemctl restart apparmor.service
```

Then rerun:

```bash
$HOME/bin/dockerd-rootless-setuptool.sh install
```

---

# 🏭 **10. Deploying to VPS (Production)**

Use **rootless Docker locally**, but on your VPS:

➡️ Install **rootful Docker** (normal Docker)
➡️ Ports 80/443 available
➡️ Reverse proxy (Traefik or Nginx Proxy Manager)
➡️ Automatic HTTPS with Let’s Encrypt
➡️ Full performance
➡️ All Docker Compose features supported

Production deployment steps:

Local machine:

```bash
docker build -t username/myapp:latest .
docker push username/myapp:latest
```

VPS:

```bash
docker pull username/myapp:latest
docker compose up -d
```

---

# 🎉 **Rootless Docker is now fully configured on Lubuntu**

You now have:

* A safe dev environment
* No root privileges needed
* Docker Compose support
* Full isolation
* Ability to build images for production
* Clear migration path to rootful Docker on VPS

