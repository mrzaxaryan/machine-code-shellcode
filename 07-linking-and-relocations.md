# Linking & Relocations

> **Series:** [executable formats](06-executable-formats.md) ← **linking** → [executable memory](08-executable-memory.md)

### What it is
Linking is the step that turns one or more **object files** (`.o`/`.obj`) plus libraries into a runnable executable (or shared library). The compiler/assembler already produced real machine code; the linker's job is to **stitch the pieces together and finalize every address**.

### Object file vs executable (the "compiled vs linked" line)
| | Object file (`.o`/`.obj`) | Executable / shared lib |
|---|---|---|
| Has real machine code? | Yes | Yes |
| Addresses final? | No — relocatable | Yes |
| Can the OS run it directly? | No | Yes |
| Created by | assembler | linker |

A compiled `.o` has the instructions, but its cross-symbol references are still placeholders.

### Relocations: "patch this address later"
When the assembler sees `call printf` but doesn't yet know where `printf` lives, it emits the call with a placeholder and records a **relocation** — a note saying "once you know `printf`'s address, patch these bytes." You can read them directly:

```
$ objdump -dr hello.o
0000000000000000 <main>:
   ...
   e8 00 00 00 00        call   5 <main+0x5>          ← placeholder bytes
              5: R_X86_64_PLT32    printf-0x4         ← relocation: type + symbol
```

That `R_X86_64_PLT32` is a relocation **type** telling the linker exactly how to compute and fill the 4 bytes (a PC-relative offset into the PLT). Common types: `R_X86_64_PC32` / `PLT32` (relative calls), `R_X86_64_64` (absolute 64-bit), `R_X86_64_GOTPCREL` (through the GOT). The linker walks every relocation, computes the final value, and writes it in.

### Static vs dynamic linking
- **Static** — the library's code is *copied into* your binary. Big, self-contained, no runtime dependencies.
- **Dynamic** — your binary keeps only *references* to shared libraries (`.so` / `.dll` / `.dylib`). A **dynamic linker/loader** (`ld.so`, Windows loader, `dyld`) resolves them at startup via tables of pointers.

### The dynamic-linking machinery: PLT/GOT and IAT
Dynamic calls can't be resolved at link time (the library's address isn't fixed until runtime). So each format keeps an indirection table the loader fills in:

```
   your call ──▶ PLT stub ──▶ GOT slot ──▶ printf's real address (resolved at runtime)
                    │              │
              jump through        initially points to a lazy resolver;
              the GOT slot        first call resolves it, then caches it
```

- **ELF** — the **PLT** (Procedure Linkage Table, trampolines) + **GOT** (Global Offset Table, addresses). Lazy binding resolves on first call.
- **PE/EXE** — the **IAT** (Import Address Table), filled by the Windows loader at startup.
- **Mach-O** — stubs + lazy/bind opcodes, resolved by `dyld`.

> **Analogy — a rolodex.** A static binary has every number written in ink. A dynamic binary has a blank rolodex (the GOT/IAT) that the loader fills in at startup by looking each number up.

> **Why it matters.** Linking is where "compiled" becomes "runnable," and relocations are the mechanism. It's also exactly the scheme freestanding and PIC code refuse — with no fixed base and no loader, there's nothing to relocate *against* (see [whole-program PIC](12-whole-program-pic.md) and [the Position-Independent Agent](19-position-independent-agent.md)).

---

▶ **Next:** The memory those linked bytes get mapped into — permissions, W^X, and obtaining an executable page — in **[executable memory](08-executable-memory.md)**.
