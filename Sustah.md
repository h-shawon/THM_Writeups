### Open Ports
```bash
rustscan -a 10.10.124.157 -- -A

Open 10.10.124.157:22
Open 10.10.124.157:80
Open 10.10.124.157:8085

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 60 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 bd:a4:a3:ae:66:68:1d:74:e1:c0:6a:eb:2b:9b:f3:33 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7zuGtMGKQdFrh6Y8Dgwdo7815klLm7VzG05KNvT112MyF41Vxz+915iRz9nTSQ583i1cmjHp+q+fMq+QGiO0iwIdYN72jop6oFxqyaO2ZjBE3grWHSP2xMsTZc7qXgPu9ZxzVAfc/4mETA8B00yc6XNApJUwfJOYz/qt/pb0WHDVBQLYesg+rrr3UZDrj9L7KNFlW74mT0nzace0yqtcV//dgOMiG8CeS6TRyUG6clbSUdr+yfgPOrcUwhTCMRKv2e30T5naBZ60e1jSuXYmQfmeZtDZ4hdsBWDfOnGnw89O9Ak+VhULGYq/ZxTh31dnWBULftw/l6saLaUJEaVeb
|   256 9a:db:73:79:0c:72:be:05:1a:86:73:dc:ac:6d:7a:ef (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBENNM4XJDFEnfvomDQgg0n7ZF+bHK+/x0EYcjrLP2BGgytEp7yg7A36KajE2QYkQKtHGPamSRLzNWmJpwzaV65w=
|   256 64:8d:5c:79:de:e1:f7:3f:08:7c:eb:b7:b3:24:64:1f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOd1NxUo0xJ3krpRI1Xm8KMCFXziZngofs/wjOkofKKV
80/tcp   open  http    syn-ack ttl 60 Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Susta
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
8085/tcp open  http    syn-ack ttl 60 Gunicorn 20.0.4
|_http-title: Spinner
| http-methods: 
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: gunicorn/20.0.4
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 5.4 (99%), Linux 3.10 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (93%), Android 5.0 - 6.0.1 (Linux 3.4) (93%), Android 5.1 (93%), Android 7.1.1 - 7.1.2 (93%)
No exact OS matches for host (test conditions non-ideal).
```

In http://10.10.117.238:8085/ I found
![[sustah1.png]]
I give an input and check the request in burpsuite. 
![[sustah2.png]]
It replies with `X-RateLimit-*` header. After googling I found [an article](https://www.linkedin.com/pulse/rate-limit-lets-bypass-manish-nemiwal/) that describes about bypassing this header. I use `X-Remote-Addr: 127.0.0.1` this in request header and it bypassed.
![[sustah4.png]]
Using ffuf I bruteforce the number.
![[sustah3.png]]
And it is `10921` that reveals the hidden directory name which is `/YouGotTh3P@th` and it was in port 80.
![[sustah5.png]]
![[sustah6.png]]
So it is Mara CMS and it is running version 7.5.
I was visiting the pages and found this.
![[sustah7.png]]
login credentials: `admin:changeme`
Searching with the CMS version I have found this:
![[sustah8.png]]
Going to this Url: http://10.10.117.238/YouGotTh3P@th/codebase/dir.php?type=filenew I uploaded a php reverse shell for listening port 5555 and obtained that shell.
![[sustah9.png]]
![[sustah10.png]]
In `/var/backups` directory I found an interesting file.
![[sustah11.png]]
Using that file I switch to user `kiran`
![[sustah12.png]]
Credentials `kiran:trythispasswordforuserkiran`
### Privilege Escalation
