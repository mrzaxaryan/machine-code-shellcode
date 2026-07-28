# Machine Code, Shellcode & Executable Formats — an Introduction

A running program is built in **layers**. This is a short tour of those layers — from the raw bytes a CPU executes, up to the app-store bundles on your phone, and then down a side path into **position-independent code**, the trick that lets machine code be *injected* and run anywhere.

The single mental model to keep in mind:

> Machine code is the **payload**. Shellcode, EXE, ELF, APK, and IPA are different **wrappers/containers** for that payload, each meant for a different *loader* (a CPU jump, the Windows PE loader, the Linux ELF loader, Android ART, iOS dyld).

## The big picture

```
   source ──compile + link──▶ machine code (the payload)
                                  │
                ┌─────────────────┼──────────────────┐
                ▼                 ▼                  ▼
            shellcode         EXE / ELF          APK / IPA
       (injected, runs       (OS-loaded,        (ZIP bundle:
        anywhere — no         native code)       DEX + ELF /
         container)                               Mach-O inside)
                                  │
                                  └─▶ what makes "runs anywhere" work?
                                        position-independent code (PIC)
```

## Read in order

1. **[Machine code](machine-code.md)** — the raw instructions a CPU executes, and how source code becomes them: compile + assemble + link, static vs dynamic linking, object file vs executable.
2. **[Executable & package formats](executable-formats.md)** — how machine code gets wrapped so an OS can load and run it: **EXE**, **ELF**, **APK**, **IPA**.
3. **[Shellcode](shellcode.md)** — the *same* machine code, repackaged for **injection** instead of an OS loader: position-independent, null-free, tiny, self-sufficient.
4. **[Position-independent code](position-independent-code.md)** — a deep dive on **PIC**, shellcode's defining constraint, and why making a *whole program* position-independent is hard.
5. **[The Position-Independent Agent](position-independent-agent.md)** — a real C++23 project that compiles an entire app into pure PIC shellcode, as a worked example of everything above.

Each article links back here and to the next, so jump in anywhere.

## Glossary

Quick reference for terms that recur across the series.

- **Opcode / operand** — an instruction's "verb" and its "arguments" in machine code (e.g. `b8 3c` = *move into `al`*, the value `3c`).
- **ISA** (Instruction Set Architecture) — the instruction vocabulary a CPU family understands: x86-64, ARM64, RISC-V, …
- **ABI / calling convention** — the rules for passing arguments, using registers and the stack, and returning values.
- **Object file** (`.o` / `.obj`) — compiled-but-not-linked machine code; addresses are still relocatable placeholders.
- **Relocation** — a "patch this address in later" note the linker uses to finalize cross-references.
- **CRT / libc** — the C runtime + standard library the linker silently adds: startup code that runs before `main` (sets up stack/heap/stdio, runs constructors), plus functions like `printf`/`malloc`. Freestanding code (`-nostdlib`) does without it.
- **PLT / GOT** — ELF's tables for resolving shared-library calls lazily, at runtime.
- **IAT** (Import Address Table) — PE/EXE's equivalent: the slots where imported DLL function addresses get filled in.
- **PIC** (Position-Independent Code) — code that runs correctly no matter where in memory it is placed.
- **Mach-O** — the macOS / iOS native executable format (the binary living *inside* an IPA).
- **DEX / ART** — Android's bytecode format and its runtime (both living *inside* an APK).
- **Loader / dynamic linker** — OS code that maps an executable into memory and resolves its dependencies (`ld.so` on Linux, the Windows loader, `dyld` on Apple).
