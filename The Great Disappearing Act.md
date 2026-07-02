In `/home/mcskidy/Documents/`
```From: mcskidy
To: whoever finds this

I had a short second when no one was watching. I used it.

I've managed to plant a few clues around the account.
If you can get into the user below and look carefully,
those three little "easter eggs" will combine into a passcode
that unlocks a further message that I encrypted in the
/home/eddi_knapp/Documents/ directory.
I didn't want the wrong eyes to see it.

Access the user account:
username: eddi_knapp
password: S0mething1Sc0ming

There are three hidden easter eggs.
They combine to form the passcode to open my encrypted vault.

Clues (one for each egg):

1)
I ride with your session, not with your chest of files.
Open the little bag your shell carries when you arrive.

2)
The tree shows today; the rings remember yesterday.
Read the ledger’s older pages.

3)
When pixels sleep, their tails sometimes whisper plain words.
Listen to the tail.

Find the fragments, join them in order, and use the resulting passcode
to decrypt the message I left. Be careful — I had to be quick,
and I left only enough to get help.

~ McSkidy
```

1. In environment variable: `PASSFRAG1=3ast3r`
2. In `/home/eddi_knapp/.secret_git` run `git log --all` to view all commit. And run `git show` to view the passfrag2: `PASSFRAG2: -1s-`
3. In `/home/eddi_knapp/Pictures/.easter_egg`: `PASSFRAG3: c0M1nG`

The passcode is: `3ast3r-1s-c0M1nG`.
In `/home/eddi_knapp/Documents` there is an encrypted file `mcskidy_note.txt.gpg`. Using this passcode decrypt the file.
`gpg -- decrypt mcskidy_note.txt.gpg > decrypt.txt`.
In `decrypt.txt`.

Now replace the content of `/home/socmas/2025/wishlist.txt` with the following that are found in decrypt.txt. 
```Form
Hardware security keys (YubiKey or similar)
Commercial password manager subscriptions (team seats)
Endpoint detection & response (EDR) licenses
Secure remote access appliances (jump boxes)
Cloud workload scanning credits (container/image scanning)
Threat intelligence feed subscription
Secure code review / SAST tool access
Dedicated secure test lab VM pool
Incident response runbook templates and playbooks
Electronic safe drive with encrypted backups
```

Now visit `localhost:8080` it will show a an encrypted message.
decrypt the message using the key that is found in decrypt.txt 
In `encrypted.txt`
```bash 
U2FsdGVkX1/7xkS74RBSFMhpR9Pv0PZrzOVsIzd38sUGzGsDJOB9FbybAWod5HMsa+WIr5HDprvK6aFNYuOGoZ60qI7axX5Qnn1E6D+BPknRgktrZTbMqfJ7wnwCExyU8ek1RxohYBehaDyUWxSNAkARJtjVJEAOA1kEOUOah11iaPGKxrKRV0kVQKpEVnuZMbf0gv1ih421QvmGucErFhnuX+xv63drOTkYy15s9BVCUfKmjMLniusI0tqs236zv4LGbgrcOfgir+P+gWHc2TVW4CYszVXlAZUg07JlLLx1jkF85TIMjQ3B91MQS+btaH2WGWFyakmqYltz6jB5DOSCA6AMQYsqLlx53ORLxy3FfJhZTl9iwlrgEZjJZjDoXBBMdlMCOjKUZfTbt3pnlHWEaGJD7NoTgywFsIw5cz7hkmAMxAIkNn/5hGd/S7mwVp9h6GmBUYDsgHWpRxvnjh0s5kVD8TYjLzVnvaNFS4FXrQCiVIcp1ETqicXRjE4T0MYdnFD8h7og3ZlAFixM3nYpUYgKnqi2o2zJg7fEZ8c=
```
Now decrypt the message Using the following command. 
```bash 
openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -salt -base64 -in encrypted.txt -out decrypted.txt -pass pass:'91J6X7R4FQ9TQPM9JX2Q9X2Z'
```

```bash
cat decrypted.txt 
```
```bash
Well done — the glitch is fixed. Amazing job going the extra mile and saving the site. Take this flag THM{w3lcome_2_A0c_2025}

NEXT STEP:
If you fancy something a little...spicier....use the FLAG you just obtained as the passphrase to unlock:
/home/eddi_knapp/.secret/dir

That hidden directory has been archived and encrypted with the FLAG.
Inside it you'll find the sidequest key.
```

Using the flag: `THM{w3lcome_2_A0c_2025}` decrypt the file `/home/eddi_knapp/.secret/dir.tar.gz.gpg`
```bash
gpg --decrypt dir.tar.gz.gpg > decrypted.tar.gz 
```
```bash 
tar -xzvf dir.tar.gz
cd dir 
ls -al 
total 420
drwxrwxr-x 2 eddi_knapp eddi_knapp   4096 Dec  1 08:25 .
drwxrwxr-x 3 eddi_knapp eddi_knapp   4096 Dec  9 01:37 ..
-rw-r--r-- 1 eddi_knapp eddi_knapp 420812 Nov 30 18:18 sq1.png
```
Copy the sq1.png file in `/home/socmas/2025/`
```bash
cp /home/eddi_knapp/.secret/dir/sq1.png /home/socmas/2025/sq1.png
```
The `sq1.png` file is: 
![[AOC_2025_1.png]]

