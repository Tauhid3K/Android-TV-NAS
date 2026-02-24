# Android TV NAS Setup 

[![License](https://img.shields.io/badge/license-MIT-green)]  
[![PHP](https://img.shields.io/badge/PHP-8.2-blue)]  
[![Termux](https://img.shields.io/badge/Termux-YES-orange)]  

> Turn your Android TV into a simple NAS server accessible via TinyFileManager or FTP.

---

## 🗂 Features

- Local NAS folder on SD card (`~/storage/external-1`)  
- Web-based access via TinyFileManager  
- Anonymous FTP access for Windows  
- Clean interface with custom folder labels  
- Supports uploads, downloads, and basic file management  

---

## 🏗 Setup Steps

### 1. Install Termux on Android TV

- Install Termux from the official source.

---

### 2. Set up storage access

```bash
termux-setup-storage


This creates:

~/storage/shared    → internal storage
~/storage/external-1 → SD card folder accessible to Termux

3. Install TinyFileManager

Copy tinyfilemanager.php into your web root (htdocs/)

Optionally create tinyfilemanager_nas.php for custom folder labels:

<?php
$root = array(
    'NAS Folder' => getenv('HOME') . '/storage/external-1'
);
$config['enable_webdav'] = true; // enable WebDAV for network access
?>

4. Start your web server
apachectl start
php-fpm


Access in browser:

http://<Android-TV-IP>/tinyfilemanager.php


Only the NAS folder is shared; other storage folders are restricted by Android permissions.

🖥 FTP Access from Windows (Anonymous)

Start the FTP server for your SD card folder:

# Run in background silently
nohup python3 -m pyftpdlib -p 2121 -w -i <Android-TV-IP> > /dev/null 2>&1 &


Replace <Android-TV-IP> with your Android TV’s local IP (example: 192.168.110.176)

Windows can connect via:

ftp://<Android-TV-IP>:2121


Choose Log on anonymously

You do not need a username or password

Auto-start FTP on boot

Add the FTP command to your Termux boot script:

nano ~/.termux/boot/start-sshd


Make the script executable:

chmod +x ~/.termux/boot/start-sshd


Now FTP will start automatically whenever Termux boots, giving Windows instant access to your SD card folder.

⚠️ Limitations

USB drives and full SD card directories are restricted by Android

Termux can only access:

~/storage/shared (internal storage)
~/storage/external-1 (SD card folder)


Full SD card access requires root or SAF permissions

🔐 Security Considerations

FTP is anonymous; only use on trusted LAN

For remote access, use HTTPS or VPN

Sensitive info like IPs are not included

📌 Summary

This setup allows you to:

Turn your Android TV into a NAS server

Access a specific folder via browser or Windows Explorer

Upload, download, and manage files safely

Keep your storage organized and secure
