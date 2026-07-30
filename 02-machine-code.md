# Machine Code

> **Series:** [intro](01-introduction.md) ← **machine code** → [assembly](03-assembly-and-encoding.md)

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

> **Analogy — publishing a book.** *Compile* translates your manuscript into the printer's language; *assemble* sets the type for each chapter with the cross-references still blank; *link* binds the chapters together, fills in every page number, and prints the finished book. Only the linked book is "runnable."

### In practice: compiling `hello.c`
You can watch the layers appear with plain commands:

```
$ gcc -c hello.c          # → hello.o   (object file: real code, addresses *relocatable*)
$ file hello.o            # "ELF 64-bit LSB relocatable, x86-64, not stripped"
$ ./hello.o               # → cannot execute: required file not found  ← can't run yet
$ gcc hello.o -o hello    # → hello     (linked executable: addresses final)
$ file hello              # "ELF 64-bit LSB pie executable, x86-64, dynamically linked"
$ ./hello                 # Hello, world!
```

`hello.o` already holds real machine code, but it can't run — its calls to `puts`/`printf` are still unresolved placeholders. Only after the linker (`gcc … -o hello`) resolves them do you get a runnable program. That object-vs-executable gap *is* the whole "compiled vs linked" distinction.

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

### The C runtime (CRT)
A runnable executable is **not just your code**. When you compile a normal C/C++ program, the linker also silently pulls in the **C runtime (CRT)** — the support layer that prepares the process *before* `main()` ever runs:

- **Startup code** — the OS loader jumps to `_start` (Linux, via glibc's `crt1.o`) or `mainCRTStartup` (Windows), *not* to `main`. It sets up the stack, builds `argc`/`argv`, initializes the heap and stdio, runs global / C++ constructors, and only then calls `main`.
- **libc** — the standard library (`printf`, `malloc`, `memcpy`, `pthread_*`…), usually shipped with — and initialized by — the CRT.
- **Shutdown** — after `main` returns, the CRT runs `exit`: flushes stdio, calls `atexit` / destructor handlers, then asks the kernel to tear the process down.

```
   OS loader maps the executable + libraries
        │
        ▼
   _start / mainCRTStartup   ◀── CRT entry point (this is NOT main)
        │  set up stack, heap, stdio; run global ctors; build argv
        ▼
   main()                    ◀── your code finally runs
        │  return
        ▼
   exit:  flush stdio → run atexit/dtors → kernel exit
```

So linking does more than resolve *your* symbols — it stitches in this whole CRT + libc layer. That's why a tiny `int main(){}` binary is still kilobytes, and why a freshly linked program "just works" with a functioning `printf` and heap.

> **Freestanding: the CRT is optional.** With `-ffreestanding -nostdlib` you opt out entirely — no startup, no libc, no `main` wrapper. Your code must supply its own entry point and reach the kernel through raw syscalls. That is exactly the regime **[shellcode](09-shellcode.md)** and the **[Position-Independent Agent](19-position-independent-agent.md)** live in: dropping the CRT is what lets them be tiny, self-contained, and free of any fixed-address dependency.

> **Why it matters.** Every "native" program — whether written in C, Rust, Go, or JIT-compiled out of Java or JavaScript — reduces to this same compiled-then-linked machine code. The formats and wrappers in the rest of the series are just different ways of *packaging* this one payload.

---

▶ **Next:** How those raw bytes are written and read — **[assembly & instruction encoding](03-assembly-and-encoding.md)**.
