# Android TV NAS Setup

[![License](https://img.shields.io/badge/license-MIT-green)]  
[![PHP](https://img.shields.io/badge/PHP-8.2-blue)]  
[![Termux](https://img.shields.io/badge/Termux-YES-orange)]

Turn your Android TV into a simple NAS server accessible via **TinyFileManager (web)** or **FTP (Windows)**.

---

## Features

- NAS folder on SD card: `~/storage/external-1`
- Web-based file access via TinyFileManager
- Anonymous FTP access for Windows
- Optional clean interface with custom folder labels
- Upload, download, and basic file management
- Optional WebDAV support via TinyFileManager

---

## Setup

### 1) Install Termux on Android TV

- Install **Termux** from an official/trusted source.

### 2) Enable storage access

Run:

```bash
termux-setup-storage
```

This creates shortcuts like:

- `~/storage/shared` → internal storage
- `~/storage/external-1` → SD card folder accessible to Termux

### 3) Install / configure TinyFileManager

1. Copy `tinyfilemanager.php` into your web root (e.g. `htdocs/`).

2. (Optional) Create a custom config file like `tinyfilemanager_nas.php` to show a nicer label and restrict browsing to only the NAS folder:

```php
<?php
$root = array(
  'NAS Folder' => getenv('HOME') . '/storage/external-1'
);

$config['enable_webdav'] = true; // optional: enable WebDAV for network access
?>
```

### 4) Start your web server

Start Apache / PHP services as needed (depending on your Termux setup):

```bash
apachectl start
php-fpm
```

Then open in a browser:

```text
http://<Android-TV-IP>/tinyfilemanager.php
```

Note: Only the NAS folder is shared. Other storage folders are restricted by Android permissions.

---

## FTP access from Windows (anonymous)

### Start an FTP server for the SD card folder

Run this (background, silent):

```bash
nohup python3 -m pyftpdlib -p 2121 -w -i <Android-TV-IP> > /dev/null 2>&1 &
```

Replace `<Android-TV-IP>` with your Android TV’s local IP (example: `192.168.110.176`).

### Connect from Windows

In File Explorer (or the Run dialog), open:

```text
ftp://<Android-TV-IP>:2121
```

Choose **Log on anonymously**. No username/password is required.

---

## Auto-start FTP on boot (Termux:Boot)

1. Create a dedicated boot script (recommended):

```bash
nano ~/.termux/boot/start-ftp
```

Paste this (edit the IP):

```bash
#!/data/data/com.termux/files/usr/bin/bash

# Start FTP in background
nohup python3 -m pyftpdlib -p 2121 -w -i <Android-TV-IP> > /dev/null 2>&1 &
```

2. Make it executable:

```bash
chmod +x ~/.termux/boot/start-ftp
```

Now the FTP server will start automatically whenever Termux boots.

---

## Limitations

Android restricts full access to USB drives and some SD card directories.

Termux can typically access:

- `~/storage/shared` (internal storage)
- `~/storage/external-1` (SD card folder)

Full SD card access usually requires **root** or **SAF permissions**.

---

## Security considerations

- FTP is **anonymous** — use only on a trusted LAN.
- For remote access, prefer **HTTPS** or a **VPN**.
- Avoid exposing FTP to the internet.

---

## Summary

With this setup you can:

- Turn your Android TV into a simple NAS
- Share a specific SD-card folder over web and FTP
- Upload/download/manage files from a browser or Windows Explorer
- Keep access limited to one folder for safety