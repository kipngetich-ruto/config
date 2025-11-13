# 🐧 **Lubuntu + Windows Dual Boot Guide**

*A complete and safe guide for installing Lubuntu alongside Windows on UEFI systems.*

---

## 📌 **Overview**

This README documents the exact steps required to correctly dual-boot **Lubuntu** and **Windows 10/11** on a modern **UEFI/GPT** system.
The guide covers:

* Preparing Windows
* Creating USB installer
* Partitioning for dual boot
* Installing Lubuntu safely
* GRUB bootloader setup
* Fixing common boot problems
* Maintaining a healthy dual-boot environment

---

## 🧩 **1. Requirements**

* Windows 10/11 already installed
* A USB flash drive (4GB+)
* Lubuntu ISO from: [https://lubuntu.me/downloads](https://lubuntu.me/downloads)
* Rufus (for creating bootable USB): [https://rufus.ie](https://rufus.ie)
* A UEFI system (most modern laptops)
* Minimum 30GB free disk space for Lubuntu

---

## 🖥 **2. Prepare Windows for Dual Boot**

### ✅ Disable Fast Startup

**Control Panel → Power Options → Choose what power buttons do → Turn off fast startup**

### ✅ (Optional) Disable BitLocker

If BitLocker is ON, disable it temporarily to avoid bootloader issues.

### ✅ Create free space for Lubuntu

Open **Disk Management**:

1. Press **Windows + X → Disk Management**
2. Identify your Windows partition (usually **C:**).
3. **Right-click → Shrink Volume**
4. Shrink by at least **30GB** (more recommended).
5. Free space will appear as **Unallocated**.

---

## 💽 **3. Create the Lubuntu Bootable USB**

Use **Rufus** with these settings:

* **Device:** your USB
* **Boot selection:** Lubuntu ISO
* **Partition scheme:** GPT
* **Target system:** UEFI (non-CSM)
* **File system:** FAT32
* Write mode: **ISO mode**

Click **Start**, wait for the USB to finish.

---

## 🚀 **4. Boot From USB**

On HP laptops:

* Restart and repeatedly press **F9** for Boot Menu
* Choose **USB Device**
* Select **Install Lubuntu**

Choose language → connect WiFi → continue.

---

## 🧭 **5. Installation Type — Choose “Something Else”**

**Never choose “Erase disk”** when dual-booting.

Partition layout should look like:

* **EFI System Partition** (FAT32, 100–500MB) → KEEP
* **Windows Partition (NTFS)** → KEEP
* **Recovery partitions** → KEEP
* **Unallocated / Free Space** → Where Linux goes

---

## 📁 **6. Create Linux Partitions**

Select **free space** and create:

### ✔ Root (`/`) Partition

* **Size:** 40–60GB
* **Type:** Primary
* **Filesystem:** ext4
* **Mount point:** `/`

### ✔ Swap Partition (optional)

* **Size:** 4–8GB
* **Use as:** swap area

### ✔ Home (`/home`) Partition

* **Size:** remaining free space
* **Type:** Primary
* **Filesystem:** ext4
* **Mount point:** `/home`

---

## 🧰 **7. Bootloader Installation (Critical)**

Set:

```
Device for bootloader installation:
    /dev/nvme0n1   (or /dev/sda)
```

**DO NOT CHOOSE a partition like /dev/nvme0n1p1.**
Always choose the **entire disk**.

This installs GRUB into the existing EFI partition and keeps Windows intact.

---

## 🛠 **8. Complete Installation**

* Click **Install Now**
* Review partitions → only Linux partitions should be formatted
* Continue
* Create your username and password
* Reboot when finished
* Remove the USB when prompted

---

# 🧵 **9. First Boot After Installation**

You should now see the **GRUB menu**:

* Lubuntu
* Windows Boot Manager

If GRUB does not appear:

* Press **ESC** → **F9**
* Choose **UEFI – Ubuntu**

---

# 🛟 **10. GRUB Fix (if system boots to a black GRUB prompt)**

Boot your Lubuntu USB → **Try Lubuntu** → open terminal.

### Identify partitions:

```bash
lsblk -f
```

### Mount your Linux system:

```bash
sudo mount /dev/nvme0n1p6 /mnt     # root partition
sudo mount /dev/nvme0n1p1 /mnt/boot/efi   # EFI partition
```

### Bind and chroot:

```bash
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt
```

### Reinstall GRUB:

```bash
grub-install /dev/nvme0n1
update-grub
exit
sudo reboot
```

GRUB will now boot correctly.

---

# 🪟 **11. Add Windows to GRUB Menu (if missing)**

In Lubuntu:

```bash
sudo nano /etc/default/grub
```

Set:

```
GRUB_DISABLE_OS_PROBER=false
```

Save → exit → update grub:

```bash
sudo update-grub
```

Windows Boot Manager will now appear.

---

# 🔧 **12. Set GRUB as Default Boot (Optional)**

In BIOS:

* Press **ESC** → **F10**
* Boot Options → UEFI Boot Order
* Move **Ubuntu** above **Windows Boot Manager**

Save & exit.

---

# 💡 **13. Tips for a Healthy Dual Boot**

* Never delete the **EFI partition**
* Never run “Reset Windows” without backing up GRUB
* Keep at least 20GB free on Windows C:
* Use **Timeshift** on Linux for snapshot backups
* Use ntfs-3g for reading Windows drives in Linux

---

# 🎉 **Dual Boot Setup Complete**

You now have:

* A stable GRUB bootloader
* Lubuntu and Windows coexisting safely
* A clean partition layout
* Repair instructions saved
* No risk of accidentally deleting Windows