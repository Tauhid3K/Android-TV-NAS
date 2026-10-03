# Android TV NAS

Turn an Android TV box into a lightweight **personal Network Attached Storage (NAS)** using Termux.

This project allows an Android TV box to provide network access to its internal storage and connected USB/SSD/HDD drives through FTP, while SSH provides remote administration.

External storage is exposed through Android's `/storage` directory, allowing multiple mounted drives to be accessed without configuring each drive individually.

> **Note:** This README covers the **NAS functionality only**. Web hosting, PHP, MariaDB, Cloudflare Tunnel, and related services are documented separately.

---

## Features

- Internal storage access over FTP
- External USB/SSD/HDD access over FTP
- Read/write access to external drives when root access is available
- Automatic detection of mounted external drives
- Multiple external drives supported
- SSH remote administration
- Automatic startup using Termux:Boot
- File transfer over a local network
- Direct media streaming over FTP
- No per-drive FTP configuration required

---

## Architecture

```text
                    Local Network
                         |
              +----------+----------+
              |                     |
          PC / Phone             TV Box
                                    |
                                Termux
                                    |
             +----------------------+----------------+
             |                      |                 |
          SSH :8022            FTP :1024         FTP :1025
                                  |                 |
                            Internal Storage    /storage
                                                    |
                                      +-------------+-------------+
                                      |             |             |
                                   USB Drive     SSD/HDD       Other Mounts
```

### Services

| Service | Port | Purpose |
|---|---:|---|
| SSH | `8022` | Remote terminal administration |
| FTP | `1024` | Internal Android/Termux storage |
| FTP | `1025` | External storage through `/storage` |

---

# 1. Requirements

## Hardware

- Android TV box
- Android device with USB support
- USB flash drive / SSD / HDD
- Local Wi-Fi or Ethernet network
- Root access for external-drive write access on devices where Android storage restrictions prevent normal applications from writing

## Software

Install:

- Termux
- Termux:Boot
- OpenSSH
- Python
- pyftpdlib

---

# 2. Install Termux

Install Termux from a trusted source such as F-Droid or the official Termux project.

After opening Termux, update the packages:

```bash
pkg update
pkg upgrade
```

Install the required packages:

```bash
pkg install openssh python
```

Install `pyftpdlib`:

```bash
pip install pyftpdlib
```

Check that Python is working:

```bash
python3 --version
```

Check pyftpdlib:

```bash
python3 -m pyftpdlib --help
```

---

# 3. Allow Termux Storage Access

Run:

```bash
termux-setup-storage
```

Android will ask for storage permission.

Allow the permission.

Termux should then create:

```text
~/storage/
```

Check it:

```bash
ls -la ~/storage
```

Typical entries include:

```text
dcim
downloads
external-1
external-2
movies
music
pictures
shared
```

The exact names depend on the Android device and mounted storage.

---

# 4. Check Mounted External Drives

Android normally exposes mounted storage under:

```text
/storage
```

Check it:

```bash
ls -la /storage
```

On a typical rooted Android TV box, you may see directories similar to:

```text
/storage/XXXXXXXX
/storage/XXXX-XXXX
/storage/emulated
/storage/self
```

The UUID-like directories normally represent removable storage devices.

For example:

```text
/storage/44603D44603D3DCC
/storage/46A4-29D7
```

The actual names will be different on other devices.

---

# 5. Internal Storage FTP Server

The first FTP server provides access to Termux's shared Android storage.

The directory used is:

```text
/data/data/com.termux/files/home/storage/shared
```

Start the FTP server:

```bash
python3 -m pyftpdlib \
-p 1024 \
-w \
-i YOUR_NAS_IP \
-d /data/data/com.termux/files/home/storage/shared \
-u YOUR_FTP_USERNAME \
-P YOUR_FTP_PASSWORD
```

### Options

| Option | Meaning |
|---|---|
| `-p 1024` | FTP port |
| `-w` | Enable write operations |
| `-i` | Network interface/IP |
| `-d` | FTP root directory |
| `-u` | FTP username |
| `-P` | FTP password |

The FTP server will then be accessible at:

```text
ftp://YOUR_NAS_IP:1024
```

---

# 6. External Storage FTP Server

For external USB/SSD/HDD storage, the FTP root can be set to:

```text
/storage
```

This allows the FTP server to expose mounted storage directories automatically.

Because Android storage permissions can restrict normal applications, root access may be required for reliable read/write access to external drives.

Start the root FTP server with:

```bash
su -c 'setsid /data/data/com.termux/files/usr/bin/python3 \
-m pyftpdlib \
-p 1025 \
-w \
-i YOUR_NAS_IP \
-d /storage \
-u YOUR_FTP_USERNAME \
-P YOUR_FTP_PASSWORD \
>/data/local/tmp/ftp1025.log 2>&1 < /dev/null &'
```

The external FTP server will then be accessible at:

```text
ftp://YOUR_NAS_IP:1025
```

---

# 7. Automatic External Drive Detection

The external FTP server uses:

```text
/storage
```

as its root directory instead of configuring a specific drive.

For example:

```text
/storage
├── DRIVE_1
├── DRIVE_2
├── emulated
└── self
```

If Android mounts another USB drive, it can appear automatically:

```text
/storage
├── DRIVE_1
├── DRIVE_2
├── DRIVE_3
├── emulated
└── self
```

No new FTP server configuration is required for each drive.

This is one of the main advantages of using `/storage` as the external FTP root.

---

# 8. FTP Directory Layout

A typical setup may look like:

```text
Android TV Box
│
├── Internal Storage
│   └── FTP :1024
│
└── /storage
    │
    ├── External Drive 1
    ├── External Drive 2
    ├── External Drive 3
    └── Other Android Mounts
```

The exact directory names depend on Android and the connected storage devices.

---

# 9. Find the TV Box IP Address

The IP address may change depending on the router's DHCP configuration.

Find the current IP:

```bash
ip -4 addr
```

For Wi-Fi:

```bash
ip -4 addr show wlan0
```

You can also check:

```bash
ip route
```

Look for an address such as:

```text
192.168.x.x
```

Use your actual address when connecting from another device.

---

# 10. Test FTP From Another Computer

From Linux:

```bash
ftp YOUR_NAS_IP 1024
```

For the external storage FTP server:

```bash
ftp YOUR_NAS_IP 1025
```

Enter the configured FTP username and password.

You can also use graphical FTP clients such as:

- FileZilla
- GNOME Files
- KDE Dolphin
- WinSCP
- Other FTP clients

---

# 11. FileZilla Configuration

Create a new connection with:

```text
Protocol: FTP
Host: YOUR_NAS_IP
Port: 1024
Encryption: Use plain FTP
Logon Type: Normal
User: YOUR_FTP_USERNAME
Password: YOUR_FTP_PASSWORD
```

For external storage, use:

```text
Port: 1025
```

The external FTP server will expose:

```text
/storage
```

as its root.

---

# 12. Linux File Manager

Many Linux desktop environments can connect directly to an FTP server.

In the file manager address bar, enter:

```text
ftp://YOUR_NAS_IP:1024
```

For external storage:

```text
ftp://YOUR_NAS_IP:1025
```

Enter your FTP credentials when requested.

---

# 13. SSH Access

SSH can be used to administer the Android TV box remotely.

Start SSH:

```bash
sshd
```

Termux's default SSH port is commonly:

```text
8022
```

Check the listening port:

```bash
ss -ltn | grep 8022
```

From another computer:

```bash
ssh -p 8022 YOUR_TERMUX_USER@YOUR_NAS_IP
```

Find the Termux username with:

```bash
whoami
```

Example:

```bash
ssh -p 8022 YOUR_TERMUX_USER@YOUR_NAS_IP
```

---

# 14. SSH Key Authentication

For better SSH security, you can use an SSH key instead of a password.

On your Linux computer:

```bash
ssh-keygen
```

Copy the public key to the TV box:

```bash
ssh-copy-id -p 8022 YOUR_TERMUX_USER@YOUR_NAS_IP
```

Then connect:

```bash
ssh -p 8022 YOUR_TERMUX_USER@YOUR_NAS_IP
```

Key-based authentication is preferable to exposing a simple password over the network.

---

# 15. Check Running NAS Services

Check SSH:

```bash
ss -ltn | grep ':8022'
```

Check internal FTP:

```bash
ss -ltn | grep ':1024'
```

Check external FTP:

```bash
su -c 'ss -ltn | grep ":1025"'
```

You can also check the FTP processes:

```bash
pgrep -af pyftpdlib
```

---

# 16. Termux:Boot Automatic Startup

To automatically start the NAS services after Android boots, install:

**Termux:Boot**

Create the boot directory:

```bash
mkdir -p ~/.termux/boot
```

Create the startup script:

```bash
nano ~/.termux/boot/start-nas
```

Use the following structure:

```bash
#!/data/data/com.termux/files/usr/bin/sh

# Keep the device awake
termux-wake-lock

# FTP credentials
FTP_USER="YOUR_FTP_USERNAME"
FTP_PASS="YOUR_FTP_PASSWORD"

# Detect current Wi-Fi IP
IP=$(ip -4 addr show wlan0 | awk '/inet / {print $2}' | cut -d/ -f1 | head -n1)

# Stop if no IP was detected
[ -z "$IP" ] && exit 1

# --------------------------------------------------
# SSH
# --------------------------------------------------

pgrep -x sshd >/dev/null 2>&1 || sshd

# --------------------------------------------------
# Internal Storage FTP
# --------------------------------------------------

if ! ss -ltn 2>/dev/null | grep -q ":1024 "; then
    nohup python3 -m pyftpdlib \
        -p 1024 \
        -w \
        -i "$IP" \
        -d "$HOME/storage/shared" \
        -u "$FTP_USER" \
        -P "$FTP_PASS" \
        >"$HOME/ftp1024.log" 2>&1 &
fi

# --------------------------------------------------
# External Storage FTP
# Requires root
# --------------------------------------------------

if ! ss -ltn 2>/dev/null | grep -q ":1025 "; then
    su -c "setsid $PREFIX/bin/python3 \
        -m pyftpdlib \
        -p 1025 \
        -w \
        -i $IP \
        -d /storage \
        -u '$FTP_USER' \
        -P '$FTP_PASS' \
        >/data/local/tmp/ftp1025.log 2>&1 < /dev/null &"
fi
```

Make the script executable:

```bash
chmod +x ~/.termux/boot/start-nas
```

---

# 17. Start NAS Manually

You can execute the boot script manually without rebooting:

```bash
~/.termux/boot/start-nas
```

Then check:

```bash
ss -ltn | grep -E ':8022|:1024|:1025'
```

Expected ports:

```text
8022   SSH
1024   Internal FTP
1025   External FTP
```

---

# 18. Reboot Test

After installing and configuring Termux:Boot:

1. Make sure Termux:Boot has been opened at least once.
2. Reboot the Android TV box.
3. Wait for Android to finish booting.
4. Open Termux if necessary.
5. Check the services from another computer.

Check:

```bash
ss -ltn | grep -E ':8022|:1024|:1025'
```

Then test:

```text
FTP :1024
FTP :1025
SSH :8022
```

---

# 19. Restart Individual FTP Servers

Stop the internal FTP server:

```bash
pkill -f 'pyftpdlib.*1024'
```

Start it again:

```bash
python3 -m pyftpdlib \
-p 1024 \
-w \
-i YOUR_NAS_IP \
-d "$HOME/storage/shared" \
-u YOUR_FTP_USERNAME \
-P YOUR_FTP_PASSWORD
```

Stop the external FTP server:

```bash
su -c 'pkill -f "pyftpdlib.*1025"'
```

Start it again:

```bash
su -c 'setsid /data/data/com.termux/files/usr/bin/python3 \
-m pyftpdlib \
-p 1025 \
-w \
-i YOUR_NAS_IP \
-d /storage \
-u YOUR_FTP_USERNAME \
-P YOUR_FTP_PASSWORD \
>/data/local/tmp/ftp1025.log 2>&1 < /dev/null &'
```

---

# 20. Streaming Media From the NAS

Because the NAS provides FTP access, media files can be accessed directly over the network.

For example:

```text
ftp://YOUR_NAS_IP:1025/DRIVE_NAME/Movies/movie.mp4
```

Applications such as VLC can open FTP media URLs.

This allows the Android TV box to act as a simple media storage server without copying the files to the client device first.

---

# 21. NAS Storage Example

Example:

```text
External Drive
│
├── Movies
│   ├── Movie 1.mkv
│   └── Movie 2.mp4
│
├── TV Shows
│   ├── Series 1
│   └── Series 2
│
├── Music
│
├── Photos
│
├── Documents
│
└── Backups
```

The exact organization is entirely up to the user.

---

# 22. Security

This project is designed primarily for use on a **trusted local network**.

## Important

FTP does **not** provide encrypted authentication or file transfer.

Therefore:

- Do not expose the FTP ports directly to the public internet.
- Do not use the same password used for important accounts.
- Use a strong, unique FTP password.
- Keep the NAS behind your router/firewall.
- Avoid port-forwarding FTP ports unless you fully understand the security implications.
- SSH should preferably use key-based authentication.
- External FTP running with root privileges should only be used when necessary.
- Limit physical access to the Android TV box and attached storage.

The placeholders in this README intentionally do not contain real credentials, IP addresses, or private identifiers.

---

# 23. Root Access and External Storage

Android storage behavior varies between devices and Android versions.

On some devices, Termux can access shared storage but cannot freely write to removable storage.

In this setup, root access is used for the external FTP server:

```text
FTP :1025
       |
       v
     /storage
       |
       +-- External Drive 1
       +-- External Drive 2
       +-- External Drive 3
```

If root access is unavailable, the external-drive FTP server may have reduced access depending on the Android device's storage permissions.

---

# 24. Check External Drive Write Access

From a root shell:

```bash
su
```

Check mounted storage:

```bash
ls -la /storage
```

Test a drive:

```bash
touch /storage/YOUR_DRIVE/test.txt
```

Check:

```bash
ls -l /storage/YOUR_DRIVE/test.txt
```

Remove the test file:

```bash
rm /storage/YOUR_DRIVE/test.txt
```

Only perform this test on a drive where you have permission to create files.

---

# 25. Troubleshooting

## FTP port is not listening

Check:

```bash
ss -ltn | grep -E ':1024|:1025'
```

Check processes:

```bash
pgrep -af pyftpdlib
```

Restart the required FTP server.

---

## External FTP cannot write files

Check root:

```bash
su
```

Then:

```bash
ls -la /storage
```

Check the drive permissions.

Also verify that the drive is mounted and not mounted read-only.

---

## External drive does not appear

Check:

```bash
ls -la /storage
```

If the drive is not present there, Android has not exposed the mount to the expected path.

Reconnect the drive and check again.

---

## SSH does not connect

Check:

```bash
ss -ltn | grep ':8022'
```

Start SSH:

```bash
sshd
```

Find the Termux username:

```bash
whoami
```

Then connect:

```bash
ssh -p 8022 YOUR_TERMUX_USER@YOUR_NAS_IP
```

---

## Find the NAS IP again

Run:

```bash
ip -4 addr show wlan0
```

or:

```bash
ip route
```

Remember that the IP address may change when using DHCP.

---

# 26. Useful Commands

### Check storage

```bash
ls -lah ~/storage
```

### Check external mounts

```bash
ls -lah /storage
```

### Check network address

```bash
ip -4 addr
```

### Check NAS ports

```bash
ss -ltn | grep -E ':8022|:1024|:1025'
```

### Check FTP processes

```bash
pgrep -af pyftpdlib
```

### Check SSH

```bash
pgrep -af sshd
```

### Check running Termux processes

```bash
ps -A
```

---

# 27. Recommended Network Setup

For normal home use:

```text
                Internet
                    |
                 Router
                    |
          +---------+---------+
          |                   |
       Computer           Android TV NAS
                              |
                    +---------+---------+
                    |                   |
                Internal            USB/SSD/HDD
                Storage              Storage
```

Keep the NAS and client devices on the same trusted LAN.

Avoid exposing:

```text
8022
1024
1025
```

directly to the internet.

---

# 28. Project Structure

A simple repository structure can be:

```text
android-tv-nas/
│
├── README.md
│
├── scripts/
│   └── start-nas
│
└── LICENSE
```

The NAS repository should contain only NAS-related configuration and documentation.

Web hosting components such as:

```text
Apache
PHP
PHP-FPM
MariaDB
Cloudflare Tunnel
Web File Manager
```

should be documented separately.

---

# 29. Example NAS Workflow

### Step 1 — Boot the TV box

Android starts.

### Step 2 — Termux:Boot starts

The NAS startup script runs.

### Step 3 — SSH starts

```text
SSH :8022
```

becomes available.

### Step 4 — Internal FTP starts

```text
FTP :1024
```

provides access to the internal shared storage.

### Step 5 — External FTP starts

```text
FTP :1025
```

provides access to:

```text
/storage
```

### Step 6 — External drives become accessible

Mounted drives appear automatically under `/storage`.

### Step 7 — Client connects

A PC, phone, or media player can connect using:

```text
ftp://YOUR_NAS_IP:1024
```

or:

```text
ftp://YOUR_NAS_IP:1025
```

---

# 30. Limitations

This is a lightweight NAS solution rather than a full commercial NAS operating system.

Possible limitations include:

- Android may change storage mounts.
- IP addresses may change when using DHCP.
- FTP traffic is unencrypted.
- External-drive write access may require root.
- Android may terminate background processes depending on device configuration.
- Performance depends on the TV box CPU, USB controller, storage device, and network.
- Wi-Fi performance may be lower than wired Ethernet.
- Power loss can interrupt file transfers.
- The TV box must remain powered on for the NAS to be available.

---

# 31. Future Improvements

Possible future additions include:

- SMB/Samba support
- SFTP support
- HTTPS-based file access
- Web-based NAS interface
- Automatic backup
- Storage monitoring
- Disk health monitoring
- User management
- Per-directory permissions
- Automatic drive detection
- Media indexing
- Download management

---

# 32. Security Reminder

Before publishing this repository, make sure the README and scripts contain **no real**:

```text
Passwords
API keys
SSH private keys
Cloud credentials
Tunnel tokens
Personal IP addresses
Private hostnames
Personal account credentials
```

Use placeholders such as:

```text
YOUR_FTP_USERNAME
YOUR_FTP_PASSWORD
YOUR_NAS_IP
YOUR_TERMUX_USER
YOUR_TUNNEL_NAME
```

Never commit real credentials to Git.

---

# License

Choose a license appropriate for your project.

For example:

```text
MIT License
```

See the `LICENSE` file for details.

---

## Summary

This project turns an Android TV box into a lightweight NAS using Termux.

```text
Android TV Box
       |
     Termux
       |
  +----+----+
  |         |
 SSH       FTP
 :8022    /   \
         /     \
      :1024   :1025
        |       |
     Internal  /storage
     Storage     |
             +---+---+
             |       |
           USB     SSD/HDD
```

The main NAS functionality is:

```text
Internal Storage  → FTP :1024
External Storage  → FTP :1025
Remote Management → SSH :8022
```

External storage is exposed through `/storage`, allowing multiple mounted drives to be accessed without creating a separate FTP configuration for each drive.
