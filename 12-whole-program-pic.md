# Whole-Program PIC

> **Series:** [data in PIC](11-data-in-position-independent-code.md) ← **whole-program PIC** → [the Position-Independent Agent](19-position-independent-agent.md)

### The gap between a stub and a program
Hand-writing a 20-byte `exit(0)` stub as PIC (the example in [shellcode](09-shellcode.md)) is easy — you control every instruction and there's barely any data. Making an **entire application** PIC is a different problem, because ordinary high-level code is full of things that *want* absolute addresses:

- **Global and static data** lives in `.data` / `.rodata` / `.bss` and is referenced by absolute address.
- **String literals** are stored in `.rodata` and pointed to.
- **Floating-point constants** are loaded from memory.
- **Function pointers** and vtables hold absolute code addresses.

A linker normally finalizes all of these against a chosen base address. To get a flat, injectable PIC blob you have to **eliminate every one of them**, turning each into the techniques from [data in PIC](11-data-in-position-independent-code.md).

### Why hand-fixing doesn't scale
You can't realistically audit and rewrite every global, string, float, and vtable in a modern C++ program by hand. The compiler emits these as a matter of course — `.rodata` strings, vtables for virtual dispatch, static initializers. A whole program is thousands of such references.

### The answer: automate it at the compiler level
The practical solution is a **compiler pass** that rewrites the program before code emission: strip the data sections, rewrite each reference position-independently, and emit a single `.text`-only blob. That's what an LLVM `pic-transform`-style pass does — covered later, in *extracting flat binaries* — and it's what makes [the Position-Independent Agent](19-position-independent-agent.md) possible.

> **Analogy — converting a whole house to modular furniture.** A 20-byte stub is like converting one shelf — easy by hand. A whole program is a house full of built-in cabinets bolted to the walls (absolute addresses). You can't unbolt each one by hand; you need a factory process (the compiler pass) that rebuilds every cabinet as freestanding, movable furniture (PIC).

> **Why it matters.** This is the bridge between "PIC is a neat trick for shellcode" and "an entire real application ships as PIC shellcode." Everything in [the Position-Independent Agent](19-position-independent-agent.md) rests on solving this at scale.

---

▶ **Next:** See it at full scale — **[the Position-Independent Agent](19-position-independent-agent.md)**, a whole app compiled to pure PIC shellcode. (Chapters 13–18 deepen the prerequisites and are forthcoming.)
