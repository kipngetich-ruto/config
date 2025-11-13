# 🧑‍💻 **Lubuntu Developer Setup Guide**

*A complete post-installation guide for development on Lubuntu.*

---

## 📌 **1. Update System**

After fresh installation:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -f
```

This ensures all system packages are up to date.

---

## 📌 **2. Essentials (Tools You Always Need)**

```bash
sudo apt install -y \
  curl \
  wget \
  git \
  build-essential \
  software-properties-common \
  ca-certificates \
  nano \
  unzip \
  htop
```

---

## 📌 **3. Install Node.js (LTS)**

Use NodeSource (recommended for stability):

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

### Verify:

```bash
node -v
npm -v
```

### Optional: install pnpm

```bash
npm install -g pnpm
```

### Optional: install Yarn

```bash
npm install -g yarn
```

---

## 📌 **4. Install GitHub SSH Key**

Generate SSH key:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Start SSH agent:

```bash
eval "$(ssh-agent -s)"
```

Add key:

```bash
ssh-add ~/.ssh/id_ed25519
```

Copy key to clipboard:

```bash
cat ~/.ssh/id_ed25519.pub
```

Add the public key to GitHub:
**GitHub → Settings → SSH and GPG keys → New key**

---

## 📌 **5. Install GitHub CLI (gh)** *(Optional but useful)*

```bash
type -p curl >/dev/null || sudo apt install curl -y
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y
```

Login:

```bash
gh auth login
```

---

## 📌 **6. Install VS Code**

### Method A — Using Microsoft Repo (recommended)

```bash
sudo apt install wget gpg -y
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'
sudo apt update
sudo apt install code -y
```

### Method B — Snap (simple)

```bash
sudo snap install code --classic
```

---

## 📌 **7. Install Python + pip + venv**

```bash
sudo apt install python3 python3-pip python3-venv -y
```

### Create a virtual environment:

```bash
python3 -m venv env
source env/bin/activate
```

---

## 📌 **8. Install Browsers**

### Chromium (open-source Chrome)

```bash
sudo apt install chromium-browser -y
```

### Google Chrome (official)

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt install ./google-chrome-stable_current_amd64.deb -y
```

---

## 📌 **9. Install Bluetooth Support**

```bash
sudo apt install blueman bluez -y
```

Launch via:
**Menu → Preferences → Bluetooth Manager**

---

## 📌 **10. Install Optional Dev Tools**

### Docker (if needed)

```bash
sudo apt install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

Add user to docker group:

```bash
sudo usermod -aG docker $USER
```

---

## 📌 **11. Install Snap (for apps)**

Lubuntu supports Snap:

```bash
sudo apt install snapd -y
```

Common snaps:

```bash
sudo snap install postman
sudo snap install slack --classic
sudo snap install discord
sudo snap install obsidian --classic
```

---

## 📌 **12. Improve Battery & Performance**

```bash
sudo apt install tlp tlp-rdw -y
sudo systemctl enable tlp
sudo systemctl start tlp
```

---

## 📌 **13. Enable Firewall**

```bash
sudo ufw enable
sudo ufw status
```

---

## 📌 **14. NTFS (Windows partitions) Auto-Mount**

Install NTFS support:

```bash
sudo apt install ntfs-3g -y
```

Auto-mount is handled automatically by Lubuntu File Manager.

---

## 📌 **15. Backup Your System (Recommended)**

Install Timeshift:

```bash
sudo apt install timeshift -y
```

Use:

* **RSYNC mode** for Linux-only partitions
* Create snapshots before major updates

---

# ✅ **Everything Installed? Verify with this command:**

```bash
node -v && npm -v && code --version && python3 --version && git --version
```

You should see all versions output successfully.

---

# 📘 **End of Developer Setup Guide**

This setup is ideal for:

* Web development (Node.js, React, Vue, Next.js, Nuxt)
* Python development
* API development
* GitHub workflows
* Daily Linux usage