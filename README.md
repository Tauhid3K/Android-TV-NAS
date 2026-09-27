# Android TV NAS Setup

Turn an Android TV box into a lightweight NAS — accessible over **FTP** (LAN),
**Tiny File Manager** (web, via LAN or Cloudflare Tunnel), and **SSH** (remote terminal).

![License](https://img.shields.io/badge/license-MIT-green)
![PHP](https://img.shields.io/badge/PHP-8.x-blue)
![Termux](https://img.shields.io/badge/Termux-Yes-orange)
![Platform](https://img.shields.io/badge/Platform-Android-lightgrey)

---

## Features

- **Internal storage — read/write** over FTP
- **External drives (USB / SSD / HDD) — read-only** over FTP
- **Tiny File Manager** web UI for browsing and downloads
- **SSH access** for remote terminal control
- **Cloudflare Tunnel** for secure remote access (no port forwarding)
- **Auto-start on boot** via Termux:Boot
- Auto-detected drives appear automatically (no per-drive setup)

---

## Requirements

- Android TV box (or any Android device) — **no root**
- [Termux](https://github.com/termux/termux-app) + [Termux:Boot](https://github.com/termux/termux-boot)
- Packages: `openssh`, `apache2`, `php`, `php-fpm`, `mariadb`, `python`, `cloudflared`
- Python package: `pyftpdlib` (`pip install pyftpdlib`)

---

## Important limitation (read this first)

Android mounts external USB/SSD/HDD volumes **read-only** for all
non-root apps — including Termux. This is enforced by the OS, not Termux.

| Location | FTP read | FTP write | Web read | Web write |
|---|---|---|---|---|
| `~/storage/shared` (internal) | ✅ | ✅ | ✅ | ✅ |
| `/storage/<UUID>` (external drive) | ✅ | ❌ | ✅ | ❌ |
| `/storage/<UUID>/Android/data/com.termux/files/` | ✅ | ✅ | ✅ | ✅ |

**Consequence:** the external-drive FTP is intentionally read-only.
For write access to external drives you need root, or an Android file-manager
app with a built-in server (Material Files, Solid Explorer, etc.).

---

## 1. Install Termux and enable storage

```bash
termux-setup-storage
```

This creates:

- `~/storage/shared` → internal storage
- `~/storage/external-1` → first SD/USB volume (symlink)

---

## 2. Install packages

```bash
pkg update && pkg upgrade
pkg install openssh apache2 php php-fpm mariadb python cloudflared
pip install pyftpdlib
```

---

## 3. Configure Tiny File Manager

1. Place `tinyfilemanager.php` in the Apache web root:
   `$PREFIX/share/apache2/default-site/htdocs/`
2. Set the root path inside the file:
   ```php
   $root_path = $_SERVER['DOCUMENT_ROOT'];
   $root_url  = '';
   $use_auth  = true;
   ```
3. Add a password:
   ```bash
   php -r "echo password_hash('YOUR_PASSWORD', PASSWORD_BCRYPT), PHP_EOL;"
   ```
   Paste the hash into `$auth_users`:
   ```php
   $auth_users = [
       'youruser' => '$2y$10$...paste-hash-here...',
   ];
   ```
4. Create one symlink so the web UI can see all storage:
   ```bash
   ln -sfn /storage $PREFIX/share/apache2/default-site/htdocs/storage
   ```

Open in a browser:

```
http://<android-tv-ip>/tinyfilemanager.php
```

---

## 4. Start FTP servers

Two servers run side by side:

| Port | Serves | Access |
|---|---|---|
| **1024** | `~/storage/shared` (internal) | read/write |
| **1025** | `/storage` (all drives) | read-only |

```bash
# Internal — writable
python -m pyftpdlib -p 1024 -w -i <android-tv-ip> \
    -d ~/storage/shared -u youruser -P yourpassword &

# External — read-only, auto-detects every plugged drive
python -m pyftpdlib -p 1025 -i <android-tv-ip> \
    -d /storage -u youruser -P yourpassword &
```

Because the second server roots at `/storage/`, any USB/SSD/HDD you plug in
appears automatically under its UUID — no watcher script, no manual linking.

---

## 5. SSH access

Termux runs an SSH server on port **8022** (non-root apps can't bind to port 22).

### 5.1 Set up SSH in Termux

```bash
# Set a password for your Termux user
passwd

# Start the SSH server (also starts automatically on boot)
sshd
```

Find your IP and username:

```bash
whoami
ifconfig | grep wlan0 -A 1
```

### 5.2 Connect from your laptop

**Linux / macOS:**

```bash
ssh -p 8022 <username>@<android-tv-ip>
```

**Windows (PowerShell or Command Prompt):**

```powershell
ssh -p 8022 <username>@<android-tv-ip>
```

Type `yes` to accept the host key on first connection, then enter the
password you set with `passwd`.

### 5.3 (Optional) SSH key login — skip the password every time

On your **laptop**, generate a key if you don't have one:

```bash
ssh-keygen -t ed25519
```

Copy it to the TV box:

```bash
ssh-copy-id -p 8022 <username>@<android-tv-ip>
```

Or manually:

```bash
cat ~/.ssh/id_ed25519.pub | ssh -p 8022 <username>@<android-tv-ip> \
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Now this works with no password:

```bash
ssh -p 8022 <username>@<android-tv-ip>
```

### 5.4 Useful SSH tips

| Task | Command |
|---|---|
| Copy a file **to** the box | `scp -P 8022 file.txt <username>@<android-tv-ip>:~/` |
| Copy a file **from** the box | `scp -P 8022 <username>@<android-tv-ip>:~/file.txt .` |
| Copy a whole folder | `scp -P 8022 -r folder/ <username>@<android-tv-ip>:~/` |
| Sync a folder (rsync) | `rsync -av -e "ssh -p 8022" folder/ <username>@<android-tv-ip>:~/` |
| Local port forward | `ssh -p 8022 -N -f -L 8080:localhost:8080 <username>@<android-tv-ip>` |

### 5.5 SSH troubleshooting

- **`Connection refused`** → run `sshd` in Termux.
- **`Permission denied`** → check username with `whoami` in Termux.
- **`Host key verification failed`** → delete the old key:
  `ssh-keygen -R "[<android-tv-ip>]:8022"`.

---

## 6. Connecting from your laptop

### 6.1 Web — Tiny File Manager

Open a browser on any device on the same Wi-Fi:

```
http://<android-tv-ip>/tinyfilemanager.php
```

Log in with the username and password you set in step 3.

### 6.2 FTP — FileZilla (recommended)

Windows Explorer caches FTP sessions and merges ports, so use
**FileZilla** for reliable access to both servers.

Download: <https://filezilla-project.org>

**Site 1 — Internal (read/write):**

| Field | Value |
|---|---|
| Protocol | FTP |
| Host | `<android-tv-ip>` |
| Port | `1024` |
| Encryption | Use plain FTP |
| Logon Type | Normal |
| User | `youruser` |
| Password | `yourpassword` |

**Site 2 — External (read-only):**

| Field | Value |
|---|---|
| Host | `<android-tv-ip>` |
| Port | `1025` |
| User | `youruser` |
| Password | `yourpassword` |

Save both — switching between them is one click.

### 6.3 FTP — Windows Explorer (works, but limited)

Explorer merges all FTP to the same host into a single session, so you
can't have both ports open at once. To use it:

1. **Control Panel → Credential Manager → Windows Credentials**
2. Remove any entry for `<android-tv-ip>`
3. Close **all** Explorer windows
4. Open a fresh window and enter in the address bar:

```
ftp://youruser:yourpassword@<android-tv-ip>:1024
```

When you want the other port, log off (right-click → **Log Off**) first,
then connect to `:1025`.

### 6.4 FTP — PowerShell (fastest test)

```powershell
ftp <android-tv-ip> 1024
```

At the `ftp>` prompt:

```
user youruser yourpassword
ls
bye
```

### 6.5 Stream video in VLC

VLC plays directly from FTP without downloading first.

**Media → Open Network Stream** → paste:

```
ftp://youruser:yourpassword@<android-tv-ip>:1025/<UUID>/path/to/file.mkv
```

Where `<UUID>` is the drive's folder name.

---

## 7. Daily usage

| I want to… | Do this |
|---|---|
| Browse files | `http://<android-tv-ip>/tinyfilemanager.php` |
| Upload a file (internal) | FileZilla → port **1024** → drag file |
| Download from external drive | FileZilla → port **1025** → drag out |
| Stream a movie | VLC → `ftp://...:1025/<UUID>/path.mkv` |
| Run a command on the box | `ssh -p 8022 <username>@<android-tv-ip>` |
| Copy a file to the box | `scp -P 8022 file <username>@<android-tv-ip>:~/` |
| Check free space | `http://<android-tv-ip>/storage_info.php` |
| Restart everything | `ssh ...` → `pkill python3; sh ~/.termux/boot/start-sshd` |

---

## 8. Auto-start on boot (Termux:Boot)

Create `~/.termux/boot/start-sshd`:

```bash
#!/data/data/com.termux/files/usr/bin/sh

sshd

pgrep php-fpm >/dev/null || php-fpm &
apachectl start
pgrep mariadbd >/dev/null || mariadbd-safe &

cloudflared tunnel run <your-tunnel-name> &

IP=$(ip -4 addr show wlan0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
termux-wake-lock

# FTP 1 — internal, writable
pgrep -f "pyftpdlib.*1024" >/dev/null || \
  nohup python3 -m pyftpdlib -p 1024 -w -i $IP \
    -d /data/data/com.termux/files/home/storage/shared \
    -u youruser -P yourpassword > /dev/null 2>&1 &

# FTP 2 — all storage, read-only
pgrep -f "pyftpdlib.*1025" >/dev/null || \
  nohup python3 -m pyftpdlib -p 1025 -i $IP \
    -d /storage \
    -u youruser -P yourpassword > /dev/null 2>&1 &
```

Make it executable:

```bash
chmod +x ~/.termux/boot/start-sshd
```

Open Termux:Boot once from your app drawer so Android registers it.

---

## 9. Cloudflare Tunnel (optional, remote access)

```bash
cloudflared tunnel login
cloudflared tunnel create <tunnel-name>
cloudflared tunnel route dns <tunnel-name> nas.example.com
cloudflared tunnel run <tunnel-name>
```

In the Cloudflare Zero Trust dashboard, add an ingress rule:

| Public hostname | Service |
|---|---|
| `nas.example.com` | `http://localhost:8080` (Apache) |

**Do not tunnel FTP** — plain FTP sends credentials in cleartext. Keep FTP LAN-only.

---

## 10. Pin the device IP (recommended)

Your TV box gets a new IP from DHCP on every reconnect, which breaks bookmarks
and tunnel configs. Fix it in your router:

- Find **DHCP Reservation / Static Lease**
- Bind the TV box's MAC to a fixed IP

---

## Repository layout

```
.
├── README.md
├── start-sshd              # Termux:Boot startup script
├── tinyfilemanager.php     # web UI (or link to upstream)
├── storage_info.php        # shows free/total space for all volumes
├── .gitignore
└── LICENSE
```

### `.gitignore`

```
*.json
.env
*.key
*.pem
*.log
```

### `storage_info.php` (optional)

Lists internal storage plus every mounted external volume by reading
`/proc/mounts`. Useful as a health-check page.

---

## Limitations

- External drives are **read-only** for non-root apps (Android scoped storage).
- Android may block writes even inside `Android/data/` on some OEM ROMs.
- FTP is unencrypted — LAN only.
- Cloudflare tunnel exposes HTTP services only; FTP is not tunneled.
- SSH on port **8022**, not 22 (Android restriction on privileged ports).

---

## Security notes

- Use a strong FTP password; the server is reachable by anything on the LAN.
- Protect Tiny File Manager with `$use_auth = true` and a bcrypt hash.
- Never expose plain FTP to the internet.
- Prefer **SSH keys** over passwords for SSH login.
- Consider a VPN (WireGuard, Tailscale) instead of opening more ports.

---

## License

MIT
