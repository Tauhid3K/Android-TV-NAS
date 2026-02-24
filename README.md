### FTP Access from Windows (Anonymous)

To access the FTP server anonymously from a Windows machine, you can use the following command:

`nohup ftp -n -v ftp://yourftpserver.com &`  

Here is an auto-start boot script example:

```bash
#!/bin/bash
service pyftpdlib start
```

This will ensure that the FTP service starts automatically at boot time.