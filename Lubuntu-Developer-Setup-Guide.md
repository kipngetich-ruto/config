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

## 📌 **2. Install Node.js (LTS)**

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

You can check on https://nodejs.org/en/download for alternative or updated option
---

Here is the **updated and improved section** for your *Lubuntu Developer Setup Guide* — now including:

✅ SSH key creation
✅ SSH key **signing** for Git commits
✅ Using **two GitHub accounts on the same PC** (perfect for Work + Personal setups)

You can copy–paste into your README.md.

---

# 🔐 **GitHub SSH Key Setup (with Signing + Multi-Account Support)**

This section configures:

* SSH authentication
* SSH commit signing
* Support for **multiple GitHub accounts** (e.g., *personal* + *work*)
* Automatic key loading
* Proper gitconfig separation

---

# 📌 **3. Generate SSH Keys (One Per GitHub Account)**

## ✔ Generate key for **Personal GitHub**

```bash
ssh-keygen -t ed25519 -C "your_personal_email@example.com" -f ~/.ssh/id_ed25519_personal
```

## ✔ Generate key for **Work GitHub**

```bash
ssh-keygen -t ed25519 -C "your_work_email@company.com" -f ~/.ssh/id_ed25519_work
```

---

# 📌 **Start SSH Agent & Add Keys**

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

To verify:

```bash
ssh-add -l
```

---

# 📌 **Add SSH Keys to GitHub**

Get your public keys:

### Personal:

```bash
cat ~/.ssh/id_ed25519_personal.pub
```

### Work:

```bash
cat ~/.ssh/id_ed25519_work.pub
```

Copy & add each to:

👉 GitHub → Settings → **SSH and GPG keys** → *New Key*
(Choose appropriate label)

---

# 📌 **Configure SSH to Use Correct Key Per Account**

Edit SSH config:

```bash
nano ~/.ssh/config
```

Add this:

```ssh
# PERSONAL GITHUB ACCOUNT
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes

# WORK GITHUB ACCOUNT
Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

Save (CTRL+O → Enter → CTRL+X)

---

# 📌 **Clone Repositories With the Correct Identity**

### 🚀 Personal Repos

```bash
git clone git@github.com-personal:username/repo.git
```

### 🚀 Work Repos

```bash
git clone git@github.com-work:company/repo.git
```

This ensures the correct SSH key is used automatically.

---

# 📌 ** Configure Git Identity Per Repository**

Inside a **personal repo**:

```bash
git config user.name "Your Name"
git config user.email "your_personal_email@example.com"
```

Inside a **work repo**:

```bash
git config user.name "Your Work Name"
git config user.email "your_work_email@company.com"
```

---

# 📌 ** Enable SSH Signing for Git Commits**

## ✔ Enable signing globally:

```bash
git config --global gpg.format ssh
git config --global commit.gpgsign true
```

## ✔ Personal repo signing:

Inside a personal repo:

```bash
git config user.signingkey ~/.ssh/id_ed25519_personal.pub
```

## ✔ Work repo signing:

Inside a work repo:

```bash
git config user.signingkey ~/.ssh/id_ed25519_work.pub
```

---

# 📌 **Add Signing Keys to GitHub**

### Personal:

Go to
**GitHub → Settings → SSH and GPG Keys → New SSH Key → Key Type: Signing Key**

Paste:

```bash
cat ~/.ssh/id_ed25519_personal.pub
```

### Work:

Same steps, but use:

```bash
cat ~/.ssh/id_ed25519_work.pub
```

---

# 📌 ** Test Signing**

In a repo:

```bash
git commit -m "test commit"
```

Then:

```bash
git log --show-signature -1
```

You should see:

```
Good "git" signature for your_name
```

---

# 📌 **Fix: Load SSH keys automatically on login**

Add to ~/.bashrc or ~/.zshrc:

```bash
eval "$(ssh-agent -s)" >/dev/null
ssh-add ~/.ssh/id_ed25519_personal 2>/dev/null
ssh-add ~/.ssh/id_ed25519_work 2>/dev/null
```

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