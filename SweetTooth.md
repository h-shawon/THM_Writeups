### Open ports
```bash
rustscan -a 10.10.203.189 -- -A

Open 10.10.203.189:111
Open 10.10.203.189:2222
Open 10.10.203.189:8086
Open 10.10.203.189:36287

PORT      STATE SERVICE REASON         VERSION
111/tcp   open  rpcbind syn-ack ttl 60
2222/tcp  open  ssh     syn-ack ttl 59 OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)
| ssh-hostkey: 
|   1024 b0:ce:c9:21:65:89:94:52:76:48:ce:d8:c8:fc:d4:ec (DSA)
| ssh-dss AAAAB3NzaC1kc3MAAACBALOlP9Bx9VQxs4JDY8vovlJp+l+pPX2MGttzN2gGNYABXAVSF9CA14OituA5tcJd5/Nv3Ru3Xyu8Yo5SV0d82rd7L/NF5Relx+iiVF+bigo329wbV3wsIrRQGUYHXiMjAs8WqQR+XKjOm3q4QLVxe/jU1I1ddy6/xO4fL7nOSh3RAAAAFQDKuQDe9pQtmnqvJkZ7QuCGm31+vQAAAIBENh/MS3oHvz1tCC4nZYwdAYZMBj2It0gYCMvD0oSkqL9IMaP9DIt/5G3D9ARrZPeSP4CqhfryIGHS7t59RNdnc3ukEsfJPo23bPBwWdIW7HXp9XDqyY1kD6L3Tq0bpeXpeXt6FQ93rFxncZngFkCrMD4+YytS532qPHMPOWh75gAAAIA7TohVech8kWTh6KIMl2Y61s9cwUqwrTkqJIYMdZ73nP69FD0bw08vyrdAwtVnsqRaNzsVVz9sBOOz3wmp/ZNI5NiuyA0UwEcxPj5k6jCn620gBpMEzVy6a8Ih3yRYHoiVMrQ/PIuoeIGxeYGckCorv8jSz2O3pq1Fnz23FRPH2A==
|   2048 7e:86:88:fe:42:4e:94:48:0a:aa:da:ab:34:61:3c:6e (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCbBmLBPg9mxkAdEbJGnz0v6Jzo4qdBcajkaIBKewKyz6OQTvyhVcDReSB2Dz0nl4mPCs3UN58hSNStCYXjZcpIBpqz2pHupVlqQ7u41Vo2W8u0nVFLt2U8JhTtA9wE6MA9GhitkN3Qorhxb3klCpSnWCDdcmkdNL0EYxZV53A52VWiNGX3vYkdMAKHAmp/VHvrsIeHozqflL8vD2UIoDmxDJwgXJRsr2iGVU1fL/Bu/DwlPwJkm50ua99yPpZbvCS9EwWki76aEtZSbcM4WHzx33Oe3tLXLCfKc9CJdIW35nBvpe5Dxl7gLR/mCHp2iTpdx1FmpSf+JjO/m2vKwL4X
|   256 04:1c:82:f6:a6:74:53:c9:c4:6f:25:37:4c:bf:8b:a8 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBHufHfqIZHVEKYC/yyNS+vTt35iULiIWoFNSQP/Bm/v90QzZjsYU9MSt7xdlR/2LZp9VWk32nl5JL65tvCMImxc=
|   256 49:4b:dc:e6:04:07:b6:d5:ab:c0:b0:a3:42:8e:87:b5 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJEYHtE8GbpGSlNB+/3IWfYRFrkJB+N9SmKs3Uh14pPj
8086/tcp  open  http    syn-ack ttl 59 InfluxDB http admin 1.3.0
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
36287/tcp open  rpcbind syn-ack ttl 60
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 5.4 (99%), Linux 3.10 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (93%), Sony Android TV (Android 5.0) (93%), Android 5.0 - 6.0.1 (Linux 3.4) (93%), Android 7.1.1 - 7.1.2 (93%)
No exact OS matches for host (test conditions non-ideal).
```
On port 8086 InfluxDB 1.3.0 is running.
Using this command I get the username: 
```bash
curl http://10.10.203.189:8086/debug/requests
{
"o5yY6yya:127.0.0.1": {"writes":2,"queries":2}
}
```
Googling the version exploit I found [this](https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933.git). Using this exploit and username I bypass the authentication and get access to the DB. 
![[sweetTooth1.png]]
From the tanks database:
![[sweetTooth9.png]]
From `water_tank` I have to find the temperature at 1621346400. Using [this website](https://www.epochconverter.com/) I get the GMT time.
![[sweetTooth7.png]]
Using that GMT time I found the temperature. That is `22.5`
![[sweetTooth6.png]]
![[sweetTooth8.png]]
Now form `mixer` database 
![[sweetTooth3.png]]
![[sweetTooth10.png]]
To get the maximum temperature.
![[sweetTooth4.png]]
Now from `creds` database
![[sweetTooth11.png]]
![[sweetTooth5.png]]
Therefore the user creds are: `uzJk6Ry98d8C:7788764472`
Using these credentials I login in ssh 
![[sweetTooth12.png]]
After enumerating I found `initializeandquery.sh` 
![[sweetTooth13.png]]
In that script
![[sweetTooth14.png]]
The `docker.sock` was writable but I had no docker binary on the machine. Here I decided to open port 8080 mentioned in the script to my local machine, where the docker binary was installed.
![[sweetTooth15.png]]
Then I connected to the docker instance and listed all the available images.
![[sweetTooth16.png]]
With the following command I spawned a shell on the instance and grabbed the flag.
![[sweetTooth17.png]]
To escape the docker container, I checked some commands, I shouldn’t be able to execute in a docker container. With the following command, I got an entry with the host linux system.
As a root user I was able to create my own directory in “/mtn” and mount the “/dev/xvda1” device.
![[sweetTooth18.png]]
