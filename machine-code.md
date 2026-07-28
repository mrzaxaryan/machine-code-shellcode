# Machine Code

> **Series:** [intro](intro.md) ← **machine code** → [shellcode](shellcode.md)

### What it is
Machine code is the binary the CPU executes directly — a sequence of **opcodes** and **operands** (`b8 3c 00 00 00` = "put 60 in `eax`"). It is:

- **Architecture-specific** (x86-64 ≠ ARM64 ≠ RISC-V). Each ISA has its own encoding.
- **Tied to an ABI/calling convention** (which registers hold args, how the stack is used).
- **What every native program ultimately reduces to**, no matter the source language (C, Rust, Go, even JIT-compiled Java/JS).

### How source becomes machine code: compile + assemble + link

```
 hello.c ──preprocess──▶ ──compile──▶ hello.s ──assemble──▶ hello.o ──link──▶ hello (executable)
 (source)                (compiler)   (asm)      (assembler)   (object)   (linker)
                          gcc/clang                              .o/.obj        ld/lld/link.exe
```

1. **Compile** — the compiler (`gcc`, `clang`, `rustc`…) translates source into **assembly** for the target ISA.
2. **Assemble** — the assembler turns assembly into an **object file** (`hello.o` / `hello.obj`). Its machine code is *relocatable*: it has the real instructions, but cross-symbol addresses are **not yet final**. They are left as placeholders plus **relocation entries** ("patch this spot with the address of `printf` later").
3. **Link** — the linker (`ld`, `lld`, `link.exe`) merges one or more object files plus libraries, resolves all symbols, assigns final addresses, and emits the runnable **executable** (or a shared library).

### Static vs dynamic linking
- **Static linking** — the library code is *copied into* your binary. Big, self-contained, no runtime dependencies.
- **Dynamic linking** — your binary keeps only *references* to shared libraries (`.so` / `.dll` / `.dylib`). A **dynamic linker/loader** (`ld.so`, Windows loader, `dyld`) resolves them at startup using tables of pointers: **PLT/GOT** (ELF), the **IAT** (PE), or stubs/lazy binds (Mach-O). Smaller binaries, shared in RAM, but depend on the right libs being present at runtime.

### Object file vs executable (the key "compiled vs linked" distinction)
| | Object file (`.o`/`.obj`) | Executable / shared lib |
|---|---|---|
| Has real machine code? | Yes | Yes |
| Addresses final? | No — relocatable | Yes |
| Can the OS run it directly? | No | Yes |
| Created by | assembler | linker |

So "compiled code" = object-file machine code with unresolved addresses; "linked code" = the final executable with everything resolved into a runnable whole.

---

▶ **Next:** The same machine code can be packaged not for an OS loader but for *injection* — that's **[shellcode](shellcode.md)**.
