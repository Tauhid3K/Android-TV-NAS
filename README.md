# Android TV NAS Setup
### FTP Access from Windows (Anonymous)

[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![PHP](https://img.shields.io/badge/PHP-8.2-blue)](https://www.php.net/)
[![Termux](https://img.shields.io/badge/Termux-YES-orange)](https://termux.com/)
To access the FTP server anonymously from a Windows machine, you can use the following command:

> Turn your Android TV into a simple NAS server accessible from anywhere using TinyFileManager.
`nohup ftp -n -v ftp://yourftpserver.com &`  

---

## Project Link

Access your NAS folder via a web browser:

> Replace with your own domain or IP if self-hosting.

---

## Features

- Local NAS folder for reading/writing files
- Web-based access via **TinyFileManager**
- Clean interface with custom folder labels
- Works with internal storage or an SD card (Termux-accessible folder)
- Supports uploads, downloads, and basic file management

---

## Setup Steps

### 1) Install Termux on Android TV

Install from the official source: https://termux.com/

### 2) Enable storage access

Run:

```bash
termux-setup-storage
```

This creates shortcuts like:

- `~/storage/shared` → internal storage
- `~/storage/external-1` → app-specific SD card folder (Termux-accessible)

### 3) Create your NAS folder
Here is an auto-start boot script example:

```bash
mkdir -p ~/nas
chmod 777 ~/nas
#!/bin/bash
service pyftpdlib start
```

### 4) Install TinyFileManager

- Copy `tinyfilemanager.php` into your web root (e.g. `htdocs/`)
- (Optional) create a separate file like `tinyfilemanager_nas.php` for custom folder labels

Example folder mapping in PHP:

```php
<?php
$root = array(
    'NAS Folder' => getenv('HOME') . '/nas'
);
?>
```

### 5) Start your web server

Example:

```bash
apachectl start
php-fpm
```

### 6) Access your NAS

Open in your browser:

- `https://your-domain.com/tinyfilemanager.php`

> Only the NAS folder is shared. Other storage folders are restricted by Android permissions.

---

## FTP Access from Windows (Anonymous)

### Start the FTP server in Termux

```bash
cd ~/storage/external-1
python3 -m pyftpdlib -p 2121 -w -i <Android-TV-IP> &
```

- `-p 2121` → FTP port
- `-w` → enable write (upload/delete)
- `<Android-TV-IP>` → your TV’s local IP (example: `192.168.110.176`)

### Connect from Windows Explorer

Open:

- `ftp://<Android-TV-IP>:2121`

Then:

- Choose **Log on anonymously**
- You do not need a username or password

### Auto-start FTP with Termux:Boot

Edit your Termux boot script:

```bash
nano ~/.termux/boot/start-sshd
```

Add at the end:

```bash
# Start FTP server for SD card folder in background
nohup python3 -m pyftpdlib -p 2121 -w -i <Android-TV-IP> &
```

Make executable:

```bash
chmod +x ~/.termux/boot/start-sshd
```

Now the FTP server starts automatically whenever Termux boots, giving Windows Explorer instant access to your SD card folder.

---

## Limitations

- USB drives and full SD card directories are restricted by Android
- Termux can only access:
  - `~/storage/shared` (internal storage)
  - `~/storage/external-1` (Termux folder on SD card)
- Full SD card access requires root or SAF permissions

---

## Security Considerations

- Do **not** expose your NAS publicly without authentication
- Use HTTPS and/or a VPN for remote access
- Avoid putting sensitive info (IPs, passwords) in the repo

---

## Summary

This setup allows you to:

- Turn your Android TV into a NAS server
- Share **one specific folder** via the web
- Upload, download, and manage files
- Keep your storage organized and more secure
This will ensure that the FTP service starts automatically at boot time.
