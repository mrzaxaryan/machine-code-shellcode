# Assembly & Instruction Encoding

> **Series:** [machine code](02-machine-code.md) ← **assembly** → [calling conventions](04-calling-conventions.md)

### What it is
Assembly is machine code's direct, human-readable form — one short mnemonic per CPU instruction, before the assembler turns it into bytes. `mov rax, 60` *is* the bytes `48 c7 c0 3c 00 00 00`; assembly just lets you write the first and read the second. A **disassembler** goes bytes → mnemonics; an **assembler** goes source → bytes.

### Anatomy of an x86-64 instruction
x86-64 is **CISC** with **variable-length** instructions (1 to 15 bytes). One instruction is a chain of mostly-optional fields:

```
   [prefixes] [opcode] [ModR/M] [SIB] [displacement] [immediate]
       │         │        │       │         │            │
       │         │        │       │         │       literal value (e.g. the 60)
       │         │        │       │      1/2/4-byte address offset
       │         │        │      scale–index–base (array indexing)
       │         │      picks operand: which register, which memory mode
       │      the "verb": what to do (mov, add, call…)
       └─ operand-size / lock / rep hints
```

Not every instruction uses every field — `cc` (`int3`) is just the opcode byte, while `48 8b 05 d8 0f 00 00` uses nearly all of them.

### A worked decode — with a PIC Easter egg
Take `48 8b 05 d8 0f 00 00`, left to right:

| bytes | field | meaning |
|---|---|---|
| `48` | REX prefix (W=1) | operand size = 64-bit |
| `8b` | opcode | `MOV r64, r/m64` (load into a register) |
| `05` | ModR/M | mod=00, reg=`rax`, r/m = **RIP-relative** |
| `d8 0f 00 00` | displacement | `+0x0fd8` |

So it decodes to `mov rax, [rip + 0xfd8]` — "load `rax` from 4056 bytes past *this instruction*." That's **RIP-relative addressing**, the cornerstone of [position-independent code](10-position-independent-code.md): the same bytes are correct no matter where they sit, because the reference is measured from the instruction itself, not a fixed address.

### Registers and addressing modes
- **General-purpose registers** — `rax, rcx, rdx, rbx, rsp, rbp, rsi, rdi, r8–r15` (64-bit; `eax` / `ax` / `al` are the lower slices).
- **Addressing modes** — register-direct (`rax`), immediate (`0x3c`), memory with displacement (`[rbp-8]`), **RIP-relative** (`[rip+disp]`), plus SIB-scaled indexing for arrays (`[rax + rbx*8]`).

### RISC vs CISC
x86 is the outlier: variable-length, with decades of accumulated encodings. **ARM and RISC-V are RISC** — every instruction is a fixed **4 bytes** with simpler, regular formats. That regularity makes them easier to decode (and to [emulate](appendix-norwx.md)), but it means PIC works differently per ISA — a topic returned to later in the series.

### Assemblers & disassemblers
- **Assemble:** `nasm -f bin`, `gcc` / `clang` (GNU `as`), `ml64`.
- **Disassemble:** `objdump -d -M intel hello`, `ndisasm -b 64`.

> **Analogy — a recipe.** Machine code is the exact gram measurements a robot chef executes; assembly is the same recipe in words a human can read. Encoding is "words → grams"; disassembly is "grams → words."

> **Why it matters.** Reading — and hand-writing — bytes is what [shellcode](09-shellcode.md) and [PIC](10-position-independent-code.md) come down to. The null-free `exit(0)` shellcode exists because someone knew that `xor eax,eax; mov al,60; syscall` encodes without null bytes.

---

▶ **Next:** How functions actually talk to each other — registers, stack, and return values — in **[calling conventions](04-calling-conventions.md)**.
