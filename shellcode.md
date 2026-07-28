# Shellcode

> **Series:** [machine code](machine-code.md) ← **shellcode** → [executable formats](executable-formats.md)

### What it is
**Shellcode** is a compact blob of machine code meant to be **injected into a target process and executed there** — not loaded by an OS as a normal file. Classically it was the payload that gave you a shell (hence the name); today it's any small native payload (reverse shell, meterpreter stager, egg hunter, etc.).

### What makes it shellcode, not just "machine code"
Shellcode is machine code with three extra constraints:

1. **Position-Independent Code (PIC)** — it must run correctly *no matter where in memory it lands*, because the attacker controls where it's copied. It uses relative addressing instead of absolute addresses and does **not** rely on a loader to fix anything up (there is no loader). We unpack exactly *how* in **[Position-Independent Code](position-independent-code.md)**.
2. **Bad-character free** — most injection paths copy bytes as a string (`strcpy`, `read` into a buffer), so a `\x00` (null) terminates the copy. Shellcode is hand-written/re-encoded to contain **no null bytes** (and often avoids `\n`, `\r`, spaces, etc., depending on the channel).
3. **Small and self-contained** — it can't assume libc is set up; it reaches the kernel directly through **syscalls** (e.g. `syscall` on x86-64, `svc 0` on ARM).

### Concrete example — Linux x86-64 "exit(0)"

Naïve (has null bytes — fine for an executable, **bad** for shellcode):
```
b8 3c 00 00 00   mov  eax, 60      ; sys_exit  (note the 00 00 00)
bf 00 00 00 00   mov  edi, 0       ; status 0
0f 05            syscall
```

Null-free shellcode version:
```
31 c0   xor  eax, eax     ; eax = 0  (clears upper bits, no nulls)
31 ff   xor  edi, edi     ; edi = 0
b0 3c   mov  al, 60       ; eax low byte = 60  → sys_exit
0f 05   syscall
```
Same result (`exit(0)`), but every byte is non-zero and it has no absolute address — it works wherever it's placed. That is the essence of shellcode.

### "Linked" shellcode
Shellcode is usually **hand-assembled** (e.g. `nasm -f bin`) so it emits raw bytes directly — no object headers, no linker. But you *can* also:

- **Compile from C with freestanding flags** (`-nostdlib -ffreestanding -fno-pic` off, PIC on, `-O2`), then use a **custom linker script** to lay everything out contiguously, and finally `objcopy -O binary` to strip headers and dump the flat `.text` as raw bytes. That flat blob is your shellcode.
- **Staged payloads**: a tiny **stage-0** loader (a few dozen bytes) does just enough — `mmap`/`VirtualAlloc` memory, mark it executable, then read or decode the larger **stage-1** payload and jump to it. This is the shellcode analog of "linking": the stage-0 stub binds the bigger payload into place at runtime.

The hard contrast: a normal executable *describes itself* to the OS (headers, sections, imports) and lets the OS place it; **shellcode places and describes nothing — it just runs where it lands.**

---

▶ **Next:** How machine code is *normally* packaged for an OS — **[executable & package formats](executable-formats.md)** (EXE / ELF / APK / IPA).
