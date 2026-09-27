# Android TV NAS Setup

Turn an Android TV box into a lightweight NAS — accessible over **FTP** (LAN)
and **Tiny File Manager** (web, via LAN or Cloudflare Tunnel).

![License](https://img.shields.io/badge/license-MIT-green)
![PHP](https://img.shields.io/badge/PHP-8.x-blue)
![Termux](https://img.shields.io/badge/Termux-Yes-orange)
![Platform](https://img.shields.io/badge/Platform-Android-lightgrey)

---

## Features

- **Internal storage — read/write** over FTP (`~/storage/shared`)
- **External drives (USB / SSD / HDD) — read-only** over FTP (`/storage`)
- **Tiny File Manager** web UI for browsing and downloads
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

### Connect from Windows

Explorer caches FTP sessions and merges ports — **use FileZilla** instead,
or PowerShell:

```powershell
ftp 192.168.1.109 1024
user youruser yourpassword
ls
```

---

## 5. Auto-start on boot (Termux:Boot)

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

## 6. Cloudflare Tunnel (optional, remote access)

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

## 7. Pin the device IP (recommended)

Your TV box gets a new IP from DHCP on every reconnect, which breaks bookmarks
and tunnel configs. Fix it in your router:

- Find **DHCP Reservation / Static Lease**
- Bind the TV box's MAC to a fixed IP (e.g. `192.168.1.109`)

---

## Repository layout

```
.
├── README.md
├── start-sshd              # Termux:Boot startup script
├── tinyfilemanager.php     # web UI (or link to upstream)
├── storage_info.php        # shows free/total space for all volumes
└── LICENSE
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

---

## Security notes

- Use a strong FTP password; the server is reachable by anything on the LAN.
- Protect Tiny File Manager with `$use_auth = true` and a bcrypt hash.
- Never expose plain FTP to the internet.
- Consider a VPN (WireGuard, Tailscale) instead of opening more ports.

---

## License

MIT
