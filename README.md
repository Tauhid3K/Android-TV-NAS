# Android TV NAS

Turn an Android TV box into a lightweight **personal NAS** using Termux.

The NAS provides network access to the Android TV box's internal storage and connected USB/SSD/HDD drives through FTP, along with SSH access for remote administration.

External drives are automatically detected through Android's `/storage` mount point, so no per-drive configuration is required.

---

## Features

- Internal storage **read/write** over FTP
- External USB/SSD/HDD **read/write** over FTP with root access
- Automatic detection of mounted external drives
- SSH remote terminal access
- Automatic startup using Termux:Boot
- File transfer over the local network
- Direct media streaming from FTP
- Multiple external drives supported simultaneously
- No per-drive configuration or watcher script required

---

## Architecture

```text
                    Android TV Box
                          │
                    ┌─────┴─────┐
                    │   Termux  │
                    └─────┬─────┘
                          │
              ┌───────────┴───────────┐
              │                       │
          FTP :1024               FTP :1025
              │                       │
              ▼                       ▼
       Internal Storage            /storage
              │                       │
              │              ┌────────┼────────┐
              │              │        │        │
              │             USB      SSD      HDD
              │
              └──────── Read/Write ────────────┘

                    SSH :8022
                       │
                       ▼
                 Remote Terminal
```

---

# Requirements

## Hardware

- Android TV box or Android device
- Wi-Fi or Ethernet connection
- USB/SSD/HDD for external NAS storage
- **Root access for external-drive write support**

Example hardware:

```text
X96 Mini P281
Amlogic S905W
2 GB RAM
Android 7.1.2
```

The setup can be adapted to other Android devices.

---

## Software

Install:

- Termux
- Termux:Boot

Required packages:

```bash
pkg update && pkg upgrade
pkg install openssh python
```

Install the FTP server:

```bash
pip install pyftpdlib
```

---

# 1. Enable Termux Storage

Run:

```bash
termux-setup-storage
```

Allow the requested Android storage permission.

Termux will create:

```text
~/storage/
├── shared
├── dcim
├── downloads
├── movies
├── music
├── pictures
└── external-*
```

The important path for internal shared storage is:

```text
~/storage/shared
```

Android's mounted storage volumes are available under:

```text
/storage/
```

For example:

```text
/storage/46A4-29D7
/storage/44603D44603D3DCC
/storage/emulated
```

The actual directory names depend on the filesystem UUIDs.

---

# 2. Check Mounted Drives

Run:

```bash
ls -la /storage
```

You may see:

```text
44603D44603D3DCC
46A4-29D7
emulated
self
```

You can also inspect mounts:

```bash
mount | grep /storage
```

Example:

```text
/dev/fuse on /storage/46A4-29D7
/dev/fuse on /storage/44603D44603D3DCC
/dev/fuse on /storage/emulated
```

---

# 3. FTP Servers

This NAS uses two FTP servers.

| Port | Location | Access |
|---|---|---|
| **1024** | Internal shared storage | Read/Write |
| **1025** | `/storage` | Read/Write with root |

---

## FTP 1024 — Internal Storage

The first FTP server exposes:

```text
~/storage/shared
```

Start it manually:

```bash
python3 -m pyftpdlib \
-p 1024 \
-w \
-i <android-tv-ip> \
-d ~/storage/shared \
-u youruser \
-P yourpassword &
```

The `-w` option enables write access.

Example:

```text
ftp://youruser@192.168.1.106:1024
```

---

# 4. FTP 1025 — External Storage

Android's storage restrictions normally prevent ordinary Termux processes from writing to removable storage.

This setup uses root to run the external FTP server.

The FTP root is:

```text
/storage
```

Start it with:

```bash
su -c 'setsid /data/data/com.termux/files/usr/bin/python3 \
-m pyftpdlib \
-p 1025 \
-w \
-i <android-tv-ip> \
-d /storage \
-u youruser \
-P yourpassword \
>/data/local/tmp/ftp1025.log 2>&1 < /dev/null &'
```

The `-w` option enables write access.

The `su` command runs the server with root privileges.

`setsid` detaches the FTP process from the Termux shell.

---

# 5. Automatic External Drive Detection

The FTP server uses:

```text
/storage
```

as its root rather than a specific drive.

Therefore, when Android mounts a new drive:

```text
/storage/<UUID>
```

the drive automatically becomes available through FTP.

For example:

```text
/storage/
├── 46A4-29D7/
├── 44603D44603D3DCC/
└── emulated/
```

Plug in another drive:

```text
/storage/
├── 46A4-29D7/
├── 44603D44603D3DCC/
├── A12B-34CD/
└── emulated/
```

No additional FTP configuration is required.

---

# 6. Example NAS Layout

```text
/storage/
│
├── 46A4-29D7/
│   ├── Movies/
│   ├── Music/
│   └── Documents/
│
├── 44603D44603D3DCC/
│   ├── Backup/
│   ├── ISO/
│   └── Videos/
│
└── emulated/
    └── 0/
        ├── DCIM/
        ├── Download/
        ├── Movies/
        └── Pictures/
```

---

# 7. SSH Access

Termux's SSH server runs on port `8022`.

Start SSH:

```bash
sshd
```

Set the Termux password:

```bash
passwd
```

Find the username:

```bash
whoami
```

Find the Wi-Fi IP:

```bash
ip -4 addr show wlan0
```

Connect from another computer:

```bash
ssh -p 8022 <username>@<android-tv-ip>
```

Example:

```bash
ssh -p 8022 u0_a59@192.168.1.106
```

---

# 8. SSH Key Authentication

Generate a key on your computer:

```bash
ssh-keygen -t ed25519
```

Copy it to the Android TV box:

```bash
ssh-copy-id -p 8022 <username>@<android-tv-ip>
```

Or manually:

```bash
cat ~/.ssh/id_ed25519.pub | \
ssh -p 8022 <username>@<android-tv-ip> \
"mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

After that:

```bash
ssh -p 8022 <username>@<android-tv-ip>
```

can be used without entering the SSH password.

---

# 9. Connecting from a PC

## FileZilla

FileZilla is recommended for FTP access.

### Internal Storage

```text
Protocol:    FTP
Host:        <android-tv-ip>
Port:        1024
Encryption:  Use plain FTP
Logon Type:  Normal
User:        youruser
Password:    yourpassword
```

### External Storage

```text
Protocol:    FTP
Host:        <android-tv-ip>
Port:        1025
Encryption:  Use plain FTP
Logon Type:  Normal
User:        youruser
Password:    yourpassword
```

---

# 10. Linux File Manager

Internal storage:

```text
ftp://youruser@<android-tv-ip>:1024
```

External storage:

```text
ftp://youruser@<android-tv-ip>:1025
```

Example:

```text
ftp://tauhid@192.168.1.106:1024
ftp://tauhid@192.168.1.106:1025
```

Enter the FTP password when requested.

---

# 11. Command-Line FTP Test

Test internal storage:

```bash
curl -u youruser:yourpassword \
ftp://<android-tv-ip>:1024/
```

Test external storage:

```bash
curl -u youruser:yourpassword \
ftp://<android-tv-ip>:1025/
```

---

# 12. Streaming Media

FTP can also be used to stream media directly.

For example, VLC can open:

```text
ftp://youruser:yourpassword@<android-tv-ip>:1025/<UUID>/Movies/movie.mkv
```

In VLC:

```text
Media → Open Network Stream
```

Paste the FTP URL.

---

# 13. Automatic Startup

Install Termux:Boot and create:

```text
~/.termux/boot/start-sshd
```

Example:

```bash
#!/data/data/com.termux/files/usr/bin/sh

# ============================================================
# Termux:Boot - Android TV NAS
# ============================================================

# Keep device awake
termux-wake-lock

# ------------------------------------------------------------
# Detect current Wi-Fi IP
# ------------------------------------------------------------
IP=$(ip -4 addr show wlan0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1)

if [ -z "$IP" ]; then
    exit 1
fi

# ------------------------------------------------------------
# SSH
# ------------------------------------------------------------
if ! pgrep -x sshd >/dev/null 2>&1; then
    sshd >/dev/null 2>&1
fi

# ------------------------------------------------------------
# FTP 1024
# Internal storage - Read/Write
# ------------------------------------------------------------
if ! ss -ltn | grep -q ":1024 "; then
    nohup python3 -m pyftpdlib \
        -p 1024 \
        -w \
        -i "$IP" \
        -d /data/data/com.termux/files/home/storage/shared \
        -u tauhid \
        -P ts1609 \
        >/data/data/com.termux/files/home/ftp1024.log 2>&1 &
fi

# ------------------------------------------------------------
# FTP 1025
# All mounted storage - ROOT + Read/Write
# ------------------------------------------------------------
if ! ss -ltn | grep -q ":1025 "; then
    su -c "setsid /data/data/com.termux/files/usr/bin/python3 \
        -m pyftpdlib \
        -p 1025 \
        -w \
        -i $IP \
        -d /storage \
        -u tauhid \
        -P ts1609 \
        >/data/local/tmp/ftp1025.log 2>&1 < /dev/null &"
fi
```

Make it executable:

```bash
chmod +x ~/.termux/boot/start-sshd
```

Open Termux:Boot at least once after installation.

The NAS services will then start automatically when Android boots.

---

# 14. Check NAS Services

Check the listening ports:

```bash
ss -ltn | grep -E ':8022|:1024|:1025'
```

Expected:

```text
:8022   SSH
:1024   Internal FTP
:1025   External FTP
```

Check FTP processes:

```bash
pgrep -af pyftpdlib
```

The root FTP process may not appear in a normal Termux process listing.

Check it as root:

```bash
su -c 'ss -ltn | grep ":1025"'
```

---

# 15. Restart NAS Services

SSH into the box:

```bash
ssh -p 8022 <username>@<android-tv-ip>
```

Then run:

```bash
~/.termux/boot/start-sshd
```

Check:

```bash
ss -ltn | grep -E ':8022|:1024|:1025'
```

Avoid manually starting another FTP server if the port is already listening.

---

# 16. Storage Permissions

Android's storage permissions vary by Android version and manufacturer.

Without root, Termux generally has:

- Full access to its own application data
- Shared storage access after `termux-setup-storage`
- Restricted access to removable storage

With root, the external FTP server can access mounted storage directly.

This project therefore uses:

```bash
su -c
```

for the external FTP server.

### Important

The external-drive write functionality depends on:

1. Root access being available
2. The Android device allowing root access to the mounted filesystem
3. The drive being mounted under `/storage`

---

# 17. Security

## FTP is unencrypted

FTP transmits credentials and data without encryption.

Do **not** expose ports `1024` or `1025` directly to the public internet.

Use FTP only on a trusted LAN.

For remote file access, prefer:

- SFTP over SSH
- VPN
- WireGuard
- Tailscale

---

## SSH

Use SSH keys instead of passwords whenever possible.

Do not expose port `8022` directly to the public internet unless properly secured.

---

## Root FTP

The external FTP server runs with root privileges.

This is powerful and potentially dangerous.

A compromised FTP account could potentially have access to files that a normal Termux process could not modify.

Use a strong FTP password and keep the service LAN-only.

---

# 18. Limitations

- External-drive write access requires root.
- Android storage behavior depends on the device and Android version.
- FTP is unencrypted.
- USB storage may disconnect or remount depending on power and hardware.
- A low-cost Android TV box is not equivalent to a dedicated NAS.
- Network speed depends on the TV box's Wi-Fi/Ethernet hardware.
- Multiple simultaneous transfers may be limited by CPU, USB, storage, and network performance.
- SSH uses port `8022` because normal Android applications cannot bind to privileged port `22`.
- External drives are identified by their Android mount/UUID directory names.

---

# 19. Recommended Network Setup

For a stable NAS, reserve the TV box's IP address in your router.

For example:

```text
Android TV box
       │
       └── DHCP reservation
               │
               ▼
        192.168.1.106
```

Then the NAS can consistently be accessed using:

```text
FTP:
ftp://<user>@192.168.1.106:1024
ftp://<user>@192.168.1.106:1025

SSH:
ssh -p 8022 <user>@192.168.1.106
```

A DHCP reservation is preferred over manually configuring a static IP inside Android.

---

# 20. Repository Layout

```text
.
├── README.md
├── start-sshd
├── storage_info.php
├── .gitignore
└── LICENSE
```

`storage_info.php` is optional and can be used as a storage health/status page if you want a lightweight storage information endpoint.

---

# 21. Example NAS Workflow

### Upload to internal storage

```text
PC
 │
 ▼
FTP :1024
 │
 ▼
~/storage/shared
```

### Upload to external SSD

```text
PC
 │
 ▼
FTP :1025
 │
 ▼
/storage/<UUID>/
 │
 ▼
External SSD
```

### Stream a movie

```text
VLC
 │
 ▼
FTP :1025
 │
 ▼
/storage/<UUID>/Movies/movie.mkv
```

### Manage the server

```text
PC
 │
 ▼
SSH :8022
 │
 ▼
Termux
```

---

# License

MIT
