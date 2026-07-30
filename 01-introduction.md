# Machine Code, Shellcode & Executable Formats — an Introduction

A running program is built in **layers**. This series walks those layers — from the raw bytes a CPU executes, up to the app-store bundles on your phone, and then deep into **position-independent code**, the trick that lets machine code be *injected* and run anywhere.

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

1. **[Machine code](02-machine-code.md)** — the raw instructions a CPU executes, and how source becomes them (compile + link).
2. **[Assembly & instruction encoding](03-assembly-and-encoding.md)** — mnemonics, opcodes, ModR/M, and decoding bytes by hand.
3. **[Calling conventions](04-calling-conventions.md)** — the ABI: where arguments go, return values, and stack discipline.
4. **[The operating system](05-operating-system.md)** — what actually *runs* your code: processes, virtual memory, syscalls.
5. **[Executable & package formats](06-executable-formats.md)** — how machine code is wrapped for an OS: EXE / ELF / APK / IPA.
6. **[Linking & relocations](07-linking-and-relocations.md)** — symbol resolution, static vs dynamic, PLT/GOT, IAT.
7. **[Executable memory](08-executable-memory.md)** — pages, R/W/X permissions, W^X / DEP-NX.
8. **[Shellcode](09-shellcode.md)** — machine code repackaged for injection: position-independent, null-free, self-sufficient.
9. **[Position-independent code](10-position-independent-code.md)** — RIP-relative addressing, GetPC, and why no loader is needed.
10. **[Data in position-independent code](11-data-in-position-independent-code.md)** — globals, strings, and floats without fixed addresses.
11. **[Whole-program PIC](12-whole-program-pic.md)** — why making an *entire application* position-independent is hard.
12. Freestanding without a runtime — the CRT, `-nostdlib`, and your own entry point. *(forthcoming)*
13. Hidden compiler dependencies — init arrays, stack canaries, unwind info, TLS. *(forthcoming)*
14. Runtime API resolution — finding APIs without libc: PEB walking and export hashing. *(forthcoming)*
15. Extracting flat binaries — linker scripts, `objcopy -O binary`, and the `pic-transform` pass. *(forthcoming)*
16. Debugging position-independent code — driving gdb/WinDbg against raw, symbol-less blobs. *(forthcoming)*
17. PIC across architectures — x86-64 RIP vs ARM `ADRP` vs RISC-V `auipc`. *(forthcoming)*
18. **[The Position-Independent Agent](19-position-independent-agent.md)** — a real C++23 project that compiles an entire app into pure PIC shellcode.

Each chapter links to the next; forthcoming ones are filled in across batches.

## Appendices

- **[Appendix A — PowerShell shellcode injector](appendix-powershell-injector.md)** — the standard W^X shellcode loader (`RW → RX`), in-process, in pure PowerShell.
- **[Appendix B — Python shellcode injector](appendix-python-injector.md)** — the same loader cross-platform: remote-process injection on Windows, `mmap`/`mprotect` on POSIX.
- **[Appendix C — NoRWX](appendix-norwx.md)** — run shellcode from non-executable memory via fault-driven emulation; no executable allocation at all.

## Glossary

Quick reference for terms that recur across the series.

- **Opcode / operand** — an instruction's "verb" and its "arguments" in machine code (e.g. `b8 3c` = *move into `al`*, the value `3c`).
- **ISA** (Instruction Set Architecture) — the instruction vocabulary a CPU family understands: x86-64, ARM64, RISC-V, …
- **ABI / calling convention** — the rules for passing arguments, using registers and the stack, and returning values.
- **Process** — a running program: a private virtual address space plus one or more threads executing its machine code.
- **Object file** (`.o` / `.obj`) — compiled-but-not-linked machine code; addresses are still relocatable placeholders.
- **Relocation** — a "patch this address in later" note the linker uses to finalize cross-references.
- **CRT / libc** — the C runtime + standard library the linker silently adds: startup code that runs before `main` (sets up stack/heap/stdio, runs constructors), plus functions like `printf`/`malloc`. Freestanding code (`-nostdlib`) does without it.
- **PLT / GOT** — ELF's tables for resolving shared-library calls lazily, at runtime.
- **IAT** (Import Address Table) — PE/EXE's equivalent: the slots where imported DLL function addresses get filled in.
- **PIC** (Position-Independent Code) — code that runs correctly no matter where in memory it is placed.
- **Virtual memory / page** — the private address space the OS gives each process, carved into fixed-size (e.g. 4 KB) pages, each tagged with R/W/X permissions.
- **DEP / NX (W^X)** — hardware memory protection: a page is writable *or* executable, not both, and data pages can't be executed. The basis for `RWX`-allocation detection.
- **Mach-O** — the macOS / iOS native executable format (the binary living *inside* an IPA).
- **DEX / ART** — Android's bytecode format and its runtime (both living *inside* an APK).
- **Loader / dynamic linker** — OS code that maps an executable into memory and resolves its dependencies (`ld.so` on Linux, the Windows loader, `dyld` on Apple).
- **Syscall** — the numbered, controlled request a user-mode program makes to ask the kernel to do something it can't on its own (touch hardware, allocate memory, read a file).
- **VEH** (Vectored Exception Handler) — a Windows callback fired on CPU exceptions (e.g. an access violation) independent of any stack frame; how [NoRWX](appendix-norwx.md) intercepts faults to emulate shellcode.
