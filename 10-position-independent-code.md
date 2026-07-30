# Position-Independent Code (PIC)

> **Series:** [shellcode](09-shellcode.md) ← **PIC** → [data in position-independent code](11-data-in-position-independent-code.md)
> **Builds on:** [shellcode](09-shellcode.md) and [machine code](02-machine-code.md).

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

This is exactly why PIC is shellcode's first constraint (the three constraints in [shellcode](09-shellcode.md)): the attacker controls where the payload lands, so the bytes must be correct there unconditionally.

Why making an *entire program* PIC is hard — and how it's automated — is covered later in [whole-program PIC](12-whole-program-pic.md).

---

▶ **Next:** How PIC code references data without fixed addresses — **[data in position-independent code](11-data-in-position-independent-code.md)**.
