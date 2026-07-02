### Open Ports 
```bash
rustscan -a 10.10.124.214 -- -A

Open 10.10.124.214:22
Open 10.10.124.214:21
Open 10.10.124.214:80
Open 10.10.124.214:501
Open 10.10.124.214:8000
Open 10.10.124.214:8080

PORT     STATE SERVICE     REASON         VERSION
21/tcp   open  ftp         syn-ack ttl 60 vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rwxr-xr-x    1 0        0             113 Sep 15  2021 password-policy.md
|_-rw-r--r--    1 0        0            1425 Sep 15  2021 ufw.status
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
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh         syn-ack ttl 60 OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a0:ef:4c:32:28:a6:4c:7f:60:d6:a6:63:32:ac:ab:27 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5/i3O28uWolhittypXr6mAEk+XOV998o/e/3wIWpGq9J1GhtGc3J4uwYpBt7SiS3mZivq9D5jgFhqhHb6zlBsQmGUnXUnQNYyqrBmGnyl4urp5IuV1sRCdNXQdt/lf6Z9A807OPuCkzkAexFUV28eXqdXpRsXXkqgkl5DCm2WEtV7yxPIbGlcmX+arDT9A5kGTZe9rNDdqzSafz0aVKRWoTHGHuqVmq0oPD3Cc3oYfoLu7GTJV+Cy6Hxs3s6oUVcruoi1JYvbxC9whexOr+NSZT9mGxDSDLS6jEMim2DQ+hNhiT49JXcMXhQ2nOYqBXLZF0OYyNKaGdgG35CIT40z
|   256 5a:6d:1a:39:97:00:be:c7:10:6e:36:5c:7f:ca:dc:b2 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHtt/3Q8agNKO48Zw3srosCs+bfCx47O+i4tBUX7VGMSpzTJQS3s4DBhGvrvO+d/u9B4e9ZBgWSqo+aDqGsTZxQ=
|   256 0b:77:40:b2:cc:30:8d:8e:45:51:fa:12:7c:e2:95:c7 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN4jv01JeDGsDfhWIJMF8HBv26FI18VLpBeNoiSGbKVp
80/tcp   open  http        syn-ack ttl 60 lighttpd 1.4.45
|_http-title: Hamlet Annotation Project
|_http-server-header: lighttpd/1.4.45
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
501/tcp  open  nagios-nsca syn-ack ttl 60 Nagios NSCA
8000/tcp open  http        syn-ack ttl 59 Apache httpd 2.4.48 ((Debian))
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache/2.4.48 (Debian)
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
8080/tcp open  http        syn-ack ttl 59 Apache Tomcat (language: en)
|_http-favicon: Spring Java Framework
| http-title: WebAnno - Log in 
|_Requested resource was http://10.10.124.214:8080/login.html
|_http-open-proxy: Proxy might be redirecting requests
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-trane-info: Problem with XML parsing of /evox/about
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Linux 4.X|2.6.X|3.X|5.X (97%)
OS CPE: cpe:/o:linux:linux_kernel:4.15 cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:5
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete

```