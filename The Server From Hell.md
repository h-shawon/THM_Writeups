Through rustscan I have found a huge amount of ports are open. So I break them in a little set of ports and perform scanning. Then I have found something interesting on port 12345. <br/>
![](Assets/tsfh1.png) <br/>
NFS share. So I go for scanning NFS share default port 111, 2049. And discovered the following. <br/>
!![](Assets/tsfh2.png) <br/>
I enumerate the share and mount it. And found backup.zip file which is locked through a password. <br/>
![](Assets/tsfh3.png) <br/>
Using john I cracked the password.  <br/>
![](Assets/tsfh4.png) <br/>
It is the home directory of user hades with the first flag and id_rsa and hint. <br/>
![](Assets/tsfh5.png) <br/>
Following the hint I again make a scan for the given port for ssh service. Found a few ports but one of them look legitimate.  <br/>
![](Assets/tsfh6.png) <br/>
Using the id_rsa file I logged in via ssh.
```bash
ssh hades@10.49.142.38 -i id_rsa
```
![](Assets/tsfh7.png) <br/>
But it give me a ruby shell. So I ran the following command found in gtfobins and got the bash shell. <br/>
```bash
exec '/bin/bash'
```
![](Assets/tsfh8.png) <br/>
Got the user flag. <br/>
## Privilege to ROOT <br/>
While enumerating found some interesting capabilities <br/>
![](Assets/tsfh9.png) <br/>
`cap_dac_read_search` allows tar to read **any file** on the system, regardless of permissions. So using this I got the root flag. <br/>
![](Assets/tsfh10.png) <br/>