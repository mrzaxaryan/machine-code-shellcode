# Position-Independent Code (PIC)

> **Series:** [shellcode](shellcode.md) ← **PIC** → [the Position-Independent Agent](position-independent-agent.md)
> **Builds on:** [shellcode](shellcode.md) and [machine code](machine-code.md).

Shellcode's defining constraint is that it must run *wherever it lands* — it is **position-independent code (PIC)**. This article slows down on that one idea: what "runs wherever it lands" actually requires of the bytes, and what it takes to make a *whole program* satisfy it.

## What it is
PIC is code that runs correctly **no matter what address it is placed at in memory**. Normal compiled code is full of absolute assumptions — "the variable is at `0x401020`", "the string is at `0x402000`", "call the function at `0x401234`". Those addresses are baked into the bytes by the linker and only work because the OS loader maps the binary at (or relocates it to) the address it expects. PIC makes no such assumption: it computes every reference **relative to itself**, so the same bytes are correct at any base address.

> **Analogy — addresses vs. directions.** Absolute addressing is "go to 123 Main St" — it only works if everyone agrees on where Main St is. PIC is "walk three doors down from *here*" — the instruction is correct no matter where "here" turns out to be, because it's measured relative to the speaker.

That difference is visible right in the bytes:

```
   absolute (position-DEPENDENT):
       mov rax, [0x402000]      ; "the data at address 0x402000"
                                 ; → breaks if the code is ever moved

   RIP-relative (position-INDEPENDENT):
       lea rax, [rip + 0x2a0]   ; "the data 0x2a0 bytes forward from here"
                                 ; → correct at any base address
```

Three mechanics make PIC work:

- **PC/RIP-relative addressing** and **relative branches** (`call rel32`, `jmp rel32`) — "jump 200 bytes forward", never "jump to `0x401234`". The offset is identical no matter where the code sits.
- **"GetPC" tricks** (e.g. `call next; next: pop rdi`) to learn an instruction's *own* address at runtime, when it needs to reference nearby data.
- **No reliance on a loader** — it fixes up nothing, because there is no loader to do so.

This is exactly why PIC is shellcode's first constraint (the three constraints in [shellcode](shellcode.md)): the attacker controls where the payload lands, so the bytes must be correct there unconditionally.

## Why whole-program PIC is hard
Hand-writing a 20-byte `exit(0)` stub as PIC (the example in [shellcode](shellcode.md)) is easy — you control every instruction. Making an **entire application** PIC is much harder, because ordinary high-level code is full of things that *want* absolute addresses:

- **Global and static data** lives in `.data` / `.rodata` / `.bss` sections and is referenced by absolute address.
- **String literals** are stored in `.rodata` and pointed to.
- **Floating-point constants** are loaded from memory.
- **Function pointers** and vtables hold absolute code addresses.

A linker normally finalizes all of these against a chosen base address. To get a flat, injectable PIC blob you have to **eliminate every one of them** — which is what the next article's project does, automatically.

---

▶ **Next:** See it done for real — **[the Position-Independent Agent](position-independent-agent.md)**, a whole app compiled to pure PIC shellcode.
