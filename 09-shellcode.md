# Shellcode

> **Series:** [executable memory](08-executable-memory.md) ← **shellcode** → [position-independent code](10-position-independent-code.md)

### What it is
**Shellcode** is a compact blob of machine code meant to be **injected into a target process and executed there** — not loaded by an OS as a normal file. Classically it was the payload that gave you a shell (hence the name); today it's any small native payload (reverse shell, meterpreter stager, egg hunter, etc.).

> **Analogy — a note thrown over a wall.** A normal program is delivered by a moving crew (the OS loader) that unpacks crates, puts every item in its assigned spot, and switches on the lights. Shellcode is a note *thrown over the wall*: it lands wherever it lands and has to make sense and run on its own — no crew, no assigned address, no setup.

### What makes it shellcode, not just "machine code"
Shellcode is machine code with three extra constraints:

1. **Position-Independent Code (PIC)** — it must run correctly *no matter where in memory it lands*, because the attacker controls where it's copied. It uses relative addressing instead of absolute addresses and does **not** rely on a loader to fix anything up (there is no loader). We unpack exactly *how* in **[Position-Independent Code](10-position-independent-code.md)**.
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

- **Compile from C with freestanding flags** (`-nostdlib -ffreestanding -fno-pic` off, PIC on, `-O2`) — opting out of the [C runtime](02-machine-code.md) entirely — then use a **custom linker script** to lay everything out contiguously, and finally `objcopy -O binary` to strip headers and dump the flat `.text` as raw bytes. That flat blob is your shellcode.
- **Staged payloads**: a tiny **stage-0** loader (a few dozen bytes) does just enough — `mmap`/`VirtualAlloc` memory, mark it executable, then read or decode the larger **stage-1** payload and jump to it. This is the shellcode analog of "linking": the stage-0 stub binds the bigger payload into place at runtime.

In practice, the C-to-shellcode path looks like:

```
$ gcc -nostdlib -ffreestanding -fPIC -O2 -c payload.c   # → payload.o
$ ld -T linker.ld payload.o -o payload.elf              # custom contiguous layout
$ objcopy -O binary payload.elf payload.bin             # strip headers → flat bytes
```

`payload.bin` is your shellcode — raw `.text`, no headers, no loader metadata. A staged payload then chains a tiny stage-0 around a bigger stage-1:

```
   target process
        │
        ▼
   stage-0 (tiny):  mmap / mprotect RWX, pull in stage-1, jump to it
        │
        ▼
   stage-1 (big):   the real payload — decoded and executed
```

> **How is shellcode actually loaded?** The standard loader allocates executable memory and jumps in — see [Appendix A (PowerShell)](appendix-powershell-injector.md) and [Appendix B (Python)](appendix-python-injector.md) for worked W^X examples (`RW → write → RX → run`). [Appendix C — NoRWX](appendix-norwx.md) goes further: run the *same* shellcode from non-executable memory, with no executable allocation at all.

The hard contrast: a normal executable *describes itself* to the OS (headers, sections, imports) and lets the OS place it; **shellcode places and describes nothing — it just runs where it lands.**

---

## Putting it together
- **Machine code** is the universal payload — architecture-specific bytes the CPU runs. *Compiling* turns source into relocatable object code; *linking* resolves it into a runnable executable.
- **Shellcode** is the *same* machine code, but repackaged for injection: position-independent, null-free, tiny, and self-sufficient.
- **EXE / ELF / Mach-O** are the *native* containers the OS loader understands — headers + sections around machine code, with dynamic-linking machinery.
- **APK / IPA** are a layer *above* that — ZIP packages that bundle a native executable (ELF inside APK, Mach-O inside IPA) with bytecode, resources, and a signature for a mobile OS / app store.

One payload, four increasingly "packaged" forms: **raw bytes → shellcode → OS executable → app store bundle.**

▶ **Next:** Deep dive on shellcode's defining constraint — **[position-independent code](10-position-independent-code.md)**.
