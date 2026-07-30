# Data in Position-Independent Code

> **Series:** [position-independent code](10-position-independent-code.md) ← **data in PIC** → [whole-program PIC](12-whole-program-pic.md)

### The problem
[PIC](10-position-independent-code.md) makes *code* position-independent with RIP-relative addressing. But real programs are mostly **data** — strings, globals, constants — and ordinary compiled data sits in `.rodata`/`.data`, referenced by absolute address. To make a blob fully PIC, every data reference has to be made position-independent too.

### Globals and statics → RIP-relative
A global becomes a RIP-relative load, just like code references:
```
   ; instead of:   mov eax, [0x402000]      ; absolute — breaks if moved
   mov eax, [rip + 0x1f4]                   ; "the global 500 bytes ahead"
```
For symbols resolved at runtime (external libraries), PIC code goes **through the GOT**: `mov rax, [rip + got_offset]` loads a pointer the dynamic linker filled in.

### Strings → built on the stack
A literal like `"cmd.exe"` normally lives in `.rodata`. PIC shellcode instead **constructs it on the stack** byte by byte, so there's no fixed-address data:
```
   mov  dword [rsp-8],  0x2e646d63   ; "cmd."   (little-endian)
   mov  dword [rsp-4],  0x00006578   ; "exe\0"
   lea  rcx, [rsp-8]                 ; rcx → "cmd.exe" living on the stack
```

### Floating-point constants → integer bitcasts
A constant like `1.5` is normally loaded from `.rodata`. PIC rewrites it as its integer bit-pattern moved into an XMM register:
```
   mov  rax, 0x3ff8000000000000      ; 1.5 as an IEEE-754 double, carried as an integer
   movq xmm0, rax                    ; reinterpreted as the float 1.5
```

### Function pointers and vtables → relative
A function pointer or vtable entry normally holds an absolute code address. PIC replaces these with **PC-relative offsets** or relative jumps, so even indirect calls stay position-independent.

> **Analogy — packing for a move.** Absolute addressing labels every box with a fixed room number ("box 12 → 0x402000"). PIC packs everything relative to *you* ("box 12 → two shelves to your left") or builds it on the spot (stack strings) — so the same boxes work no matter which house you're in.

> **Why it matters.** These are exactly the rewrites an automated compiler pass applies to an entire program — and why a whole app can become PIC without hand-writing every instruction (see [whole-program PIC](12-whole-program-pic.md) and [the Position-Independent Agent](19-position-independent-agent.md)).

---

▶ **Next:** Why applying all of this to a *whole program* is hard — **[whole-program PIC](12-whole-program-pic.md)**.
