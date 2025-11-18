
# 📘 **PostgreSQL Encrypted Backup & Restore Documentation**

**Environment:** VPS (PostgreSQL) + On-Prem Machine (Decryption/Archival)
**Method:** GPG Public-Key Encryption + Rsync Pull
**Security Level:** Production-grade, Offline Private Key, Zero Trust VPS

---

# 📌 Overview

This document explains how to set up a **secure, encrypted backup pipeline** for a PostgreSQL database running on a VPS. The VPS generates **daily GPG-encrypted backups** using **public-key encryption only**, and an on-prem machine **pulls** the backups for long-term storage and decryption.

**Key Security Model:**

* VPS **only has the GPG public key**
* Your **on-prem machine holds the private key**
* VPS **cannot decrypt** backups (even if hacked)
* Backups pulled to your local machine remain **end-to-end encrypted**
* Only *you* can decrypt backups

---

# 🧩 Architecture

```
    VPS (Public Key Only)
       |  
       | 1) Daily pg_dump → gzip → GPG encrypt
       v
 /var/backups/postgresql/*.sql.gz.gpg
       |
       | 2) Offsite pull (rsync/ssh)
       v
  On-Prem Machine (Private Key)
       |
       | 3) Optional cloud archival (B2/S3/NAS)
       v
 Long-term encrypted storage
```

---

# 🔐 1. Generate GPG Keys (On Your Local Machine)

## Create GPG keypair locally:

```bash
gpg --full-generate-key
```

Choose:

* RSA 4096
* Never expires
* Strong passphrase

## Export the **private key** (store offline!):

```bash
gpg --armor --export-secret-key "your-email" > private-backup-key.asc
chmod 600 private-backup-key.asc
```

## Export the **public key**:

```bash
gpg --armor --export "your-email" > public-backup-key.asc
```

This will be uploaded to the VPS.

---

# 📥 2. Import Public Key on VPS

Upload:

```bash
scp public-backup-key.asc root@YOUR_VPS_IP:/tmp/
```

On the VPS:

```bash
gpg --import /tmp/public-backup-key.asc
```

Verify:

```bash
gpg --list-keys
```

Ensure there is **no secret key**:

```bash
gpg --list-secret-keys
```

Output should be empty.

---

# 🗄️ 3. Create Backup Directory on VPS

```bash
sudo mkdir -p /var/backups/postgresql
sudo chmod 700 /var/backups/postgresql
sudo chown root:root /var/backups/postgresql
```

---

# 📝 4. VPS Backup Script

File:

`/usr/local/bin/pgbackup.sh`

```bash
#!/bin/bash

###########################################
# VPS DAILY BACKUP (GPG PUBLIC KEY ONLY)
###########################################

BACKUP_DIR="/var/backups/postgresql"
LOGFILE="/var/log/pgbackup.log"
RETENTION_DAYS=7
DATE=$(date +\%F-%H%M)
BACKUP_FILE="$BACKUP_DIR/backup-$DATE.sql.gz.gpg"
GPG_KEY="your-email-or-key-id"

mkdir -p "$BACKUP_DIR"

echo "--------------------------------------" >> "$LOGFILE"
echo "[$(date)] Backup started" >> "$LOGFILE"

###########################################
# 1. PG DUMP → GZIP → GPG ENCRYPT
###########################################

if sudo -u postgres pg_dumpall | gzip | \
   gpg --batch --yes --trust-model always \
   -r "$GPG_KEY" -e > "$BACKUP_FILE"; then

    echo "[$(date)] Backup encrypted: $BACKUP_FILE" >> "$LOGFILE"
else
    echo "[$(date)] ERROR: Backup encryption failed!" >> "$LOGFILE"
    exit 1
fi

###########################################
# 2. LIGHTWEIGHT INTEGRITY CHECK
###########################################

FILESIZE=$(stat -c%s "$BACKUP_FILE")
if [ "$FILESIZE" -lt 50000 ]; then
    echo "[$(date)] ERROR: Backup file too small: $FILESIZE bytes" >> "$LOGFILE"
else
    echo "[$(date)] Size OK: $FILESIZE bytes" >> "$LOGFILE"
fi

if gpg --list-packets "$BACKUP_FILE" > /dev/null 2>&1; then
    echo "[$(date)] GPG packet structure OK" >> "$LOGFILE"
else
    echo "[$(date)] ERROR: GPG structure invalid!" >> "$LOGFILE"
fi

###########################################
# 3. ROTATION
###########################################

find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -name "*.gpg" -delete
echo "[$(date)] Removed old backups" >> "$LOGFILE"

echo "[$(date)] Backup completed" >> "$LOGFILE"
```

Make executable:

```bash
chmod +x /usr/local/bin/pgbackup.sh
```

---

# ⏰ 5. Enable Daily Cron Job on VPS

```bash
sudo crontab -e
```

Add:

```
0 2 * * * /usr/local/bin/pgbackup.sh
```

Runs automatically every night at 2 AM.

---

# 🛰️ 6. On-Prem Pull Script (Offsite Backup)

File:

`~/pull-vps-backups.sh`

```bash
#!/bin/bash

###############################################
# OFFSITE BACKUP PULL SCRIPT (ZSH + CRON SAFE)
###############################################

VPS_USER="root"
VPS_HOST="YOUR_VPS_IP"
VPS_DIR="/var/backups/postgresql"

LOCAL_DIR="/home/YOUR_USER/vps-backups"
LOGFILE="/home/YOUR_USER/pull-backup.log"

mkdir -p "$LOCAL_DIR"

######## LOAD SSH AGENT (for cron) ########

if [ -z "$SSH_AUTH_SOCK" ]; then
    for sock in /tmp/ssh-*/agent.*; do
        if [ -S "$sock" ]; then
            export SSH_AUTH_SOCK="$sock"
            break
        fi
    done

    if [ -z "$SSH_AUTH_SOCK" ]; then
        eval "$(ssh-agent -s)"
        ssh-add /home/YOUR_USER/.ssh/id_ed25519
    fi
fi

echo "----------------------------------" >> "$LOGFILE"
echo "[ $(date) ] Pulling backups..." >> "$LOGFILE"

rsync -avz --ignore-existing \
   $VPS_USER@$VPS_HOST:$VPS_DIR/*.gpg \
   "$LOCAL_DIR/" >> "$LOGFILE" 2>&1

find "$LOCAL_DIR" -type f -mtime +60 -name "*.gpg" -delete

echo "[ $(date) ] Done." >> "$LOGFILE"
```

Make executable:

```bash
chmod +x ~/pull-vps-backups.sh
```

---

# ⏲️ 7. Cron Job on Local Machine

```bash
crontab -e
```

Add:

```
0 3 * * * /bin/bash /home/YOUR_USER/pull-vps-backups.sh
```

---

# 🔓 8. Decrypting a Backup (Local Machine)

Pick a backup file:

```
backup-2025-11-20.sql.gz.gpg
```

Decrypt:

```bash
gpg --decrypt backup.sql.gz.gpg | gunzip > backup.sql
```

---

# 🗃️ 9. Restoring PostgreSQL from a Backup

Upload `backup.sql` to a VPS / new VPS:

```bash
scp backup.sql root@new_vps:/tmp/
```

On VPS:

```bash
sudo -u postgres psql < /tmp/backup.sql
```

Database restored.

---

# 🧪 10. Test Restore Locally (Recommended)

Create a test DB:

```bash
createdb test_restore
```

Restore:

```bash
psql test_restore < backup.sql
```

Drop:

```bash
dropdb test_restore
```

---

# 🛡️ 11. Security Notes

* **Never store private key on VPS**
* **Encrypt backups with public key only**
* **Decrypt only offline**
* **Backup the private key securely**
* **Automate pulling, never pushing**
* **Do periodic test restores**

This is an industry-standard secure backup model.

---

# 🧯 12. Troubleshooting

### VPS asks for private key?

❌ You mistakenly imported secret key on VPS.
Fix with:

```bash
gpg --delete-secret-keys "your-email"
```

---

### Cron not running backup?

Check:

```bash
grep CRON /var/log/syslog
```

---

### Cron not pulling backups (local)?

Probably SSH agent issue.
Check:

```bash
ssh-add -l
```
