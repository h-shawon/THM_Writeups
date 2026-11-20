# When Hearts Collide — Matchmaker (MD5 Collision)

## 1. Reconnaissance

### 1.1 The landing page

```bash
curl -s http://10.49.190.158/
```

The homepage spells out the matching logic verbatim:

> "Matchmaker keeps up with the universe by comparing your photo's **MD5 hash**
> to every doggo snapshot... If your hash is identical to a pup's, that's our
> cue... the algorithm wags its glittering tail and declares you a match."

So: **match condition = `md5(my_upload) == md5(some_stored_dog)`**.

It also links to a sample stored dog: `/view/00795a8b-fb58-47c0-91be-af068ddc71b4`.

### 1.2 The upload flow

```bash
echo "hello" > test.txt
curl -s -i -F "file=@test.txt" http://10.49.190.158/upload
```

- `POST /upload` → `302` redirect to `/upload_success/<uuid>`.
- Files are stored at `/static/uploads/<uuid>.jpg` and served back.
- **No file‑type validation** — a 6‑byte text file was accepted and stored as
  `.jpg`. This matters: our exploit payloads do **not** need to be valid images.

The success page carries the verdict in an HTML attribute:

```html
<div class="match-result" aria-live="polite" data-match="false">
```

`data-match="true"` ⇒ match (flag); `data-match="false"` ⇒ no match.

### 1.3 The de‑duplication guard (the key observation)

Re‑uploading an existing image unchanged does **not** produce a match. Download
the sample dog and send its exact bytes back:

```bash
curl -s http://10.49.190.158/static/uploads/00795a8b-fb58-47c0-91be-af068ddc71b4.jpg -o dog_sample.jpg
curl -s -i -F "file=@dog_sample.jpg" http://10.49.190.158/upload   # follow the Location
```

The result page is titled **"Matchmaker / Already uploaded"**:

> "Your photo already lives here — we already received that **exact snapshot**."

So the app has two separate checks:

| Check | What it compares | Trigger |
|-------|------------------|---------|
| "Already uploaded" guard | the **raw file bytes** ("exact snapshot") | byte‑identical file |
| "Match" decision | the **MD5 hash** | equal MD5 |

These two checks use different notions of "the same file." That gap is the
vulnerability: **a file can be byte‑different (passes the guard) yet
MD5‑identical (triggers the match).** That is precisely an MD5 collision.

---

## 2. The vulnerability — why MD5 breaks this

MD5 is not collision‑resistant. Since Wang & Yu (2004) and Marc Stevens'
`fastcoll`/HashClash work, anyone can generate two distinct byte strings `A ≠ B`
with `md5(A) == md5(B)` in under a second on a laptop.

Attack plan:

1. Obtain a collision pair `A` and `B`: `A ≠ B` but `md5(A) == md5(B) == X`.
2. **Upload `A`.** `X` is new to the database → stored as a "dog." No match yet.
3. **Upload `B`.** `B`'s bytes differ from `A`, so the *"already uploaded"*
   byte‑comparison guard does **not** fire. But the match engine computes
   `md5(B) == X`, which now equals the MD5 of the just‑stored dog `A`
   → **match → flag.**

The two "hearts" (photos) **collide** on their MD5. 🫀🫀

---

## 3. Exploitation

### 3.1 Getting a collision pair

The fastest way is a purpose‑built generator such as **`fastcoll`** (identical‑prefix
MD5 collision) from Marc Stevens' HashClash:

```bash
# Build fastcoll (if not packaged):
git clone https://github.com/cr-marcstevens/hashclash
cd hashclash && cmake . && make          # produces src/md5fastcoll/fastcoll
# Generate two colliding files from any prefix:
fastcoll -o collA.bin collB.bin          # collA != collB, md5 equal
```

Because the target does no image validation, **any** collision pair works. For a
fully self‑contained PoC (no tools to install) I used the classic, publicly
documented **Wang et al. MD5 collision** — two 128‑byte blocks that share
MD5 `79054025255fb1a26e4bc422aef54eb4`:

```bash
# Two 128-byte messages, identical MD5, differ in a handful of bytes.
cat > a.hex <<'EOF'
d131dd02c5e6eec4693d9a0698aff95c2fcab58712467eab4004583eb8fb7f8955ad340609f4b30283e488832571415a085125e8f7cdc99fd91dbdf280373c5bd8823e3156348f5bae6dacd436c919c6dd53e2b487da03fd02396306d248cda0e99f33420f577ee8ce54b67080a80d1ec69821bcb6a8839396f9652b6ff72a70
EOF
cat > b.hex <<'EOF'
d131dd02c5e6eec4693d9a0698aff95c2fcab50712467eab4004583eb8fb7f8955ad340609f4b30283e4888325f1415a085125e8f7cdc99fd91dbd7280373c5bd8823e3156348f5bae6dacd436c919c6dd53e23487da03fd02396306d248cda0e99f33420f577ee8ce54b67080280d1ec69821bcb6a8839396f965ab6ff72a70
EOF
xxd -r -p a.hex > collA.bin
xxd -r -p b.hex > collB.bin
```

Verify the paradox — **same MD5, different files**:

```bash
md5sum    collA.bin collB.bin   # both: 79054025255fb1a26e4bc422aef54eb4
sha256sum collA.bin collB.bin   # DIFFERENT -> genuinely distinct files
cmp       collA.bin collB.bin   # "differ: byte 20" -> not byte-identical
```

Output:

```
79054025255fb1a26e4bc422aef54eb4  collA.bin
79054025255fb1a26e4bc422aef54eb4  collB.bin
8d12236e5c4ed9f4e790db4d868fd5c3...  collA.bin   <- sha256 differs
b9fef2a8fc93b05e7701e97196fda6c4...  collB.bin
collA.bin collB.bin differ: byte 20, line 1
```

### 3.2 Firing the exploit

```bash
# Step 1 — upload A: seeds the hash X into the "doggy database"
curl -s -i -F "file=@collA.bin" http://10.49.190.158/upload | grep -i location
# -> /upload_success/<uuidA>   (page shows data-match="false")

# Step 2 — upload B: same MD5 X, different bytes -> MATCH
LOC=$(curl -s -i -F "file=@collB.bin" http://10.49.190.158/upload | awk 'tolower($1)=="location:"{print $2}' | tr -d '\r')
curl -s "http://10.49.190.158$LOC" | grep -iE 'data-match|THM\{'
```

Result:

```html
<div class="match-result" aria-live="polite" data-match="true">
  <span class="match-flag"><i class="fa fa-dog"></i> THM{hash_puppies_4_all}</span>
```

The first upload (`collA.bin`) returned `data-match="false"`; the second
(`collB.bin`) returned `data-match="true"` and the flag.

### 3.3 One‑liner reproduction

```bash
cd $(mktemp -d)
printf '%s' 'd131dd02c5e6eec4693d9a0698aff95c2fcab58712467eab4004583eb8fb7f8955ad340609f4b30283e488832571415a085125e8f7cdc99fd91dbdf280373c5bd8823e3156348f5bae6dacd436c919c6dd53e2b487da03fd02396306d248cda0e99f33420f577ee8ce54b67080a80d1ec69821bcb6a8839396f9652b6ff72a70' | xxd -r -p > A
printf '%s' 'd131dd02c5e6eec4693d9a0698aff95c2fcab50712467eab4004583eb8fb7f8955ad340609f4b30283e4888325f1415a085125e8f7cdc99fd91dbd7280373c5bd8823e3156348f5bae6dacd436c919c6dd53e23487da03fd02396306d248cda0e99f33420f577ee8ce54b67080280d1ec69821bcb6a8839396f965ab6ff72a70' | xxd -r -p > B
H=http://10.49.190.158
curl -s -F file=@A $H/upload >/dev/null
L=$(curl -s -i -F file=@B $H/upload | awk 'tolower($1)=="location:"{print $2}' | tr -d '\r')
curl -s "$H$L" | grep -oE 'THM\{[^}]+\}'
```

---

## 4. Flag

```
THM{hash_puppies_4_all}
```
