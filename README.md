# laptop_disaster
# Iron Server Playbook

---

# 📜 Preface: The Journey

This project started with a simple plan: Ned wanted to add a freshly built Ubuntu laptop to his existing setup — a basement Ubuntu server and a separate basement Windows development machine — by tunneling into Windows via SSH and RDP.

Initially, the idea was just to improve remote access with minimal disruption.

But as Ned worked, he realized that with a full Linux install on the laptop, there was potential to go much bigger:

- The laptop could be used as a true remote management terminal.
- The aging, inefficient basement server could be **consolidated**.
- A new, leaner, faster, more centralized **Iron Server** could replace two machines.

From that simple goal grew a complete rebuild: flattening hardware, migrating and recovering critical data, creating a full Linux server environment, virtualizing Windows, running a Bitcoin node, and modernizing network file sharing.

As always — this is Ned we are talking about. It was never going to stay simple.

This playbook documents the full journey.

---

# 👨‍💻 Overview

**Goal:** Flatten multiple old machines (Windows, old Ubuntu server), rebuild a unified, powerful Linux server that:

- Runs Ubuntu 24.04 LTS
- Runs a clean Windows 11 VM (inside virt-manager)
- Runs a Bitcoin Full Node (daemon mode)
- Shares large file storage with Windows VMs via Samba
- Operates lean, clean, redundant, and fast

---

# ✅ Phase 0: Laptop Setup for Remote Management

### 1. Flatten and Install Ubuntu Desktop 24.04 LTS
- Fresh install on the laptop.
- Full disk wipe (if necessary).
- Standard desktop install (no server edition).

### 2. Install Essential Packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim curl wget git net-tools htop glances remmina samba-client
```

### 3. Setup SSH Access to Iron Server
```bash
sudo apt install -y openssh-server
ssh <your-username>@<your-server-ip>
```

### 4. Setup Remote Desktop Access to Windows VM
- Enable RDP in Windows 11 guest.
- Connect via Remmina.

---

# ✅ Phase 1: Rebuild Physical Server (Baseline Ubuntu Install)

### 1. Flatten Machine and Install Ubuntu
- Partition: 1G /boot/efi, 2G /boot, Rest for LVM (root).

### 2. Post-Install Essentials
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim curl wget git net-tools htop glances samba nfs-common
```

### 3. Enable Virtualization
```bash
sudo apt install -y virt-manager qemu-kvm libvirt-daemon-system bridge-utils
sudo usermod -aG libvirt,libvirt-qemu <your-username>
sudo systemctl enable libvirtd --now
```

### 4. Disable Firewall Temporarily (fix SMB, migration issues)
```bash
sudo ufw disable
```

---

# ✅ Phase 2: Bring Old Storage Online

### 1. List Drives
```bash
lsblk -o NAME,SIZE,MOUNTPOINT
```

### 2. Mount Old NVMe Read-Only
```bash
sudo mkdir -p /mnt/old_nvme
sudo mount -o ro /dev/nvme1n1p3 /mnt/old_nvme
```

### 3. Backup Critical Data
```bash
sudo mkdir -p /media/<your-username>/<your-drive-uuid>/backups/old_server
sudo rsync -avh /mnt/old_nvme/home/ /media/<your-username>/<your-drive-uuid>/backups/old_server/home/
```

### ⚡ Special Note: MBR Damage and Recovery
- Accidentally corrupted both Windows VHD and Linux server MBRs.
- **Always mount old drives read-only first.**
- Used `qemu-nbd` for safe VHD inspection.
- Rsynced data before any recovery attempts.

---

# ✅ Phase 3: Build Windows 11 VM

### 1. Prepare Storage
```bash
sudo wipefs -a /dev/nvme1n1
sudo parted -a optimal /dev/nvme1n1 -- mklabel gpt
sudo parted -a optimal /dev/nvme1n1 -- mkpart primary ntfs 0% 100%
sudo mkfs.ntfs -f /dev/nvme1n1p1
```

### 2. Create Windows VM
- Use virt-manager.
- CPU: 6 cores
- RAM: 10 GB
- Storage: qcow2 or raw file on new partition.

---

# ✅ Phase 4: Setup Bitcoin Full Node

### 1. Install Bitcoin Core
```bash
wget https://bitcoincore.org/bin/bitcoin-core-28.1/bitcoin-28.1-x86_64-linux-gnu.tar.gz
sudo tar -xvzf bitcoin-28.1-x86_64-linux-gnu.tar.gz -C /usr/local --strip-components=1
```

### 2. Create Configuration File
```bash
nano /media/<your-username>/<your-drive-uuid>/BTC/bitcoin.conf
```

Example bitcoin.conf:
```bash
datadir=/media/<your-username>/<your-drive-uuid>/BTC
prune=0
server=1
listen=1
rpcuser=<your-username>
rpcpassword=yourpassword
rpcallowip=127.0.0.1
rpcport=8332
```

### 3. Create Systemd Service
```bash
sudo nano /etc/systemd/system/bitcoin-node.service
```

Example bitcoin-node.service:
```ini
[Unit]
Description=Bitcoin Daemon
After=network.target

[Service]
ExecStart=/usr/local/bin/bitcoind -datadir=/media/<your-username>/<your-drive-uuid>/BTC
User=<your-username>
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable bitcoin-node
sudo systemctl start bitcoin-node
```

### 4. Wallet Management
```bash
bitcoin-cli -datadir=/media/<your-username>/<your-drive-uuid>/BTC getwalletinfo
```

---

# ✅ Phase 5: Setup Samba Shares for Windows VM

### 1. Install Samba
```bash
sudo apt install samba
```

### 2. Edit smb.conf
```ini
[windowsbackup]
   path = /media/<your-username>/<your-drive-uuid>/F/windows_files
   browsable = yes
   writable = yes
   guest ok = yes
   public = yes
   force user = <your-username>
```

### 3. Restart and Allow Firewall
```bash
sudo systemctl restart smbd nmbd
sudo ufw allow samba
```

### 4. Create Samba User
```bash
sudo smbpasswd -a <your-username>
```

### 5. Map Network Drive in Windows
```cmd
net use Z: \\<your-server-ip>\windowsbackup /user:<your-username> yourpassword
```

---

# 📊 Lessons Learned

- Always mount old drives read-only first.
- Always validate file shares with `smbclient -L localhost -N`.
- Always open firewall ports immediately after Samba setup.
- Remember case sensitivity for Linux usernames in Samba.
- Separate Bitcoin daemon from Bitcoin-Qt frontend cleanly.
- Maintain tight, modular configurations.

---

# 🏆 Final Grade: **A+ Professional Home Lab Rebuild**

---

# 🛠️ Future Enhancements

- Automate Bitcoin node monitoring with a dashboard.
- Mirror important shares to external backup disks.
- Improve firewall rules with tighter zoning.

---

# ✨ END OF PLAYBOOK
