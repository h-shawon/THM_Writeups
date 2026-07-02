### Open Ports
```bash
rustscan -a 10.10.157.107 -- -A 

Open 10.10.157.107:21
Open 10.10.157.107:22
Open 10.10.157.107:80

PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 60 ProFTPD 1.3.5a
22/tcp open  ssh     syn-ack ttl 60 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 44:2f:fb:3b:f3:95:c3:c6:df:31:d6:e0:9e:99:92:42 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDtgGI2Qpv+ora/iClEVeJSyw673ED4ciilMWv/Cw2NtVl9oB8A5rKktZYnJDw5sYZOvXimjb20Rk6a742anZZA87PM3StTZy8ZAMDEwdt8omaz5zy1c+HcJi4jjUIzPAZK10iKJ0JnyZ3eZZgEXALsU1zTi6U8Wn+6pixB9yRzAV8FVd/UThmC8vkiyNbNJUF6tgP+paajOIq2KzcmYrn8zZFL79EjDUUqSx72/wc/VUYyNArVGtVmOuvW1TBQwnpUv3zNQL1sabfiRzmgWB4unfHCVbj8autfHOfHSpMxC5QOuOJRTdhak6MUlHbjSXBF5MU1OP4mNTIoh/+e8k17
|   256 92:24:36:91:7a:db:62:d2:b9:bb:43:eb:58:9b:50:14 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCE8pJD7f5qX4X2kInnJf/m5wbTLOFA3I49Hyi2MrHxg3jREHseTbpqk00Xmy7F2+8Z8ljTdJwD9aafUAPgXxes=
|   256 34:04:df:13:54:21:8d:37:7f:f8:0a:65:93:47:75:d0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPxHqNM/ISBztZhs47D+flKJiTqFqt5kJrFDoeNyO8Zb
80/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Welcome To Becon Mental Hospital
|_http-server-header: Apache/2.4.18 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5.4
OS details: Linux 5.4
```

I have visited the web page
![[pb1.png]]
I the source code page
![[pb2.png]]
I went to that `/sadistRoom` directory
![[pd3.png]]
Clicking that button I found the locker key.
![[pd4.png]]
Using that locker key I access the `/lockerRoom` page.
![[pd5.png]]
There I found a cipher text that was needed to be decoded for the map.
![[pd6.png]]
Using cyberchef I decode the text. And use it to access to map.
![[pd7.png]]
![[pd8.png]]
![[pd9.png]]
First I have visited the safe heaven and in the source code
![[pd10.png]]
Using ffuf I bruteforce directory and found this:
```bash
ffuf -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.10.157.107/SafeHeaven/FUZZ -t 100
```
![[pd16.png]]
in the `/keeper` page
![[pd11.png]]
Clicking that button
![[pd12.png]]
There I reverse search the given image and get the location.
![[pd13.png]]
And received the Keeper key.
![[pd14.png]]
In abandoned room I use that key. 
![[pd15.png]]
And came here.
![[pd17.png]]
By clicking that button.
![[pd18.png]]
In the source code I found
![[pd19.png]]
Maybe there is a command injection vulnerability.
![[pd20.png]]
And yes!
Using `ls ..` command I found.
![[pd21.png]]
I went to the other directory. 
![[pd22.png]]
I downloaded the zip file and got this 
![[pd23.png]]
I converted the jpg file into zip the extract it
![[pd24.png]]
The key.wav file is a morse code file.
Using [this site](https://morsecode.world/international/decoder/audio-decoder-adaptive.html) I decoded the audio. Which is `SHOWME`.
Using this I decoded the `Joseph_Oda.jpg`
![[pd25.png]]
So the ftp credentials are: `joseph:intotheterror445`
Using that credentials I loged in to the ftp server and downloaded the files.
![[pd26.png]]
I use the following python code to bruteforce the program
![[pd27.png]]
and the answer
![[pd28.png]]
```bash
kidman => Correct

Well Done !!!
Decode This => 55 444 3 6 2 66 7777 7 2 7777 7777 9 666 777 3 444 7777 7777 666 7777 8 777 2 66 4 33
```
To identify the cipher i have used decode.fr
![[pd29.png]]
After decoding I got:
![[pd30.png]]
Using this ssh credentials `kidman:KIDMANSPASSWORDISSOSTRANGE` I logged in to the server 
![[pd31.png]]
I found pkexec is SUID bit set so I use [this](https://github.com/rvizx/CVE-2021-4034.git) and became root.
![[pd32.png]]
