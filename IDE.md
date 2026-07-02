### Open ports 
```bash
rustscan -a 10.10.85.95 -- -A 

Open 10.10.85.95:21
Open 10.10.85.95:22
Open 10.10.85.95:80
Open 10.10.85.95:62337

PORT      STATE SERVICE REASON         VERSION
21/tcp    open  ftp     syn-ack ttl 60 vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.17.17.146
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp    open  ssh     syn-ack ttl 60 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e2:be:d3:3c:e8:76:81:ef:47:7e:d0:43:d4:28:14:28 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC94RvPaQ09Xx+jMj32opOMbghuvx4OeBVLc+/4Hascmrtsa+SMtQGSY7b+eyW8Zymxi94rGBIN2ydPxy3XXGtkaCdQluOEw5CqSdb/qyeH+L/1PwIhLrr+jzUoUzmQil+oUOpVMOkcW7a00BMSxMCij0HdhlVDNkWvPdGxKBviBDEKZAH0hJEfexz3Tm65cmBpMe7WCPiJGTvoU9weXUnO3+41Ig8qF7kNNfbHjTgS0+XTnDXk03nZwIIwdvP8dZ8lZHdooM8J9u0Zecu4OvPiC4XBzPYNs+6ntLziKlRMgQls0e3yMOaAuKfGYHJKwu4AcluJ/+g90Hr0UqmYLHEV
|   256 a8:82:e9:61:e4:bb:61:af:9f:3a:19:3b:64:bc:de:87 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBzKTu7YDGKubQ4ADeCztKu0LL5RtBXnjgjE07e3Go/GbZB2vAP2J9OEQH/PwlssyImSnS3myib+gPdQx54lqZU=
|   256 24:46:75:a7:63:39:b6:3c:e9:f1:fc:a4:13:51:63:20 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ+oGPm8ZVYNUtX4r3Fpmcj9T9F2SjcRg4ansmeGR3cP
80/tcp    open  http    syn-ack ttl 60 Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
62337/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-favicon: Unknown favicon MD5: B4A327D2242C42CF2EE89C623279665F
|_http-title: Codiad 2.8.4
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Adtran 424RG FTTH gateway (93%), Linux 2.6.32 (93%), Linux 3.11 (93%), Linux 3.2 - 4.9 (93%), Linux 3.5 (93%)
```

ftp allows anonymous login 
![[ide1.png]]
view the content of the file
```bash
more -- -                                    
Hey john,
I have reset the password as you have asked. Please use the default password to login. 
Also, please take care of the image file ;)
- drac.
```
visit the url http://<machine_ip:62337>
![[ide2.png]]
the CMS is 
`Codiad version 2.8.4`
According to the ftp, credentials are 
`username: john password:password`
Using searchsploit I have found a exploit for this version
 ![[ide3.png]]
 Using the first exploit I get the shell
 ![[ide4.png]]
### Privilege Escalation 
Using the SUID to pkexec get the root privilege.
![[ide5.png]]
 Now retrieve all the flags.
 ![[ide6.png]]
 