# Android TV NAS Setup :

[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![PHP](https://img.shields.io/badge/PHP-8.2-blue)](https://www.php.net/)
[![Termux](https://img.shields.io/badge/Termux-YES-orange)](https://termux.com/)

> Turn your Android TV into a simple NAS server accessible from anywhere using TinyFileManager.

---

## 🌟 Project Link

Access your NAS folder via web browser:

[https://tauhidshahriar.xyz/tinyfilemanager.php](https://tauhidshahriar.xyz/tinyfilemanager.php)

> Replace with your own domain or IP if self-hosting.

---

## 🗂 Features

- Local NAS folder for reading/writing files  
- Web-based access via **TinyFileManager**  
- Clean interface with custom folder labels  
- Works with internal storage or SD card folder  
- Supports uploads, downloads, and basic file management  

---

## 🏗 Setup Steps

### 1. Install Termux on Android TV

Install from the official source: [Termux](https://termux.com/)

### 2. Set up storage access

```bash
termux-setup-storage
This creates:

~/storage/shared → internal storage

~/storage/external-1 → app-specific SD card folder

3. Create your NAS folder
mkdir -p ~/nas
chmod 777 ~/nas

4. Install TinyFileManager

Copy tinyfilemanager.php into your web root (htdocs/)

Optionally create tinyfilemanager_nas.php for custom folder labels

Example folder mapping in PHP:

<?php
$root = array(
    'NAS Folder' => getenv('HOME') . '/nas'
);
?>

5. Start your web server
apachectl start
php-fpm

6. Access your NAS

Open in your browser:

https://your-domain.com/tinyfilemanager.php


Only the NAS folder is shared. Other storage folders are restricted by Android permissions.

⚠️ Limitations

USB drives and full SD card directories are restricted by Android

Termux can only access:

~/storage/shared (internal storage)

~/storage/external-1 (Termux folder on SD card)

Full SD card access requires root or SAF permissions

🔐 Security Considerations

Do not expose your NAS publicly without authentication

Use HTTPS or VPN for remote access

Sensitive info like IPs and passwords are not included

📌 Summary

This setup allows you to:

Turn your Android TV into a NAS server

Access a specific folder via web from anywhere

Upload, download, and manage files safely

Keep your storage organized and secure
