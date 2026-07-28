# Machine Code, Shellcode & Executable Formats — an Introduction

A running program is built in **layers**. This is a short tour of those layers — from the raw bytes a CPU executes, up to the app-store bundles on your phone, and then down a side path into **position-independent code**, the trick that lets machine code be *injected* and run anywhere.

The single mental model to keep in mind:

> Machine code is the **payload**. Shellcode, EXE, ELF, APK, and IPA are different **wrappers/containers** for that payload, each meant for a different *loader* (a CPU jump, the Windows PE loader, the Linux ELF loader, Android ART, iOS dyld).

## Read in order

1. **[Machine code](machine-code.md)** — the raw instructions a CPU executes, and how source code becomes them: compile + assemble + link, static vs dynamic linking, object file vs executable.
2. **[Shellcode](shellcode.md)** — the *same* machine code, repackaged for **injection** instead of an OS loader: position-independent, null-free, tiny, self-sufficient.
3. **[Executable & package formats](executable-formats.md)** — how machine code gets wrapped so an OS can load and run it: **EXE**, **ELF**, **APK**, **IPA**.
4. **[Position-independent code](position-independent-code.md)** — a deep dive on **PIC**, shellcode's defining constraint, and why making a *whole program* position-independent is hard.
5. **[The Position-Independent Agent](position-independent-agent.md)** — a real C++23 project that compiles an entire app into pure PIC shellcode, as a worked example of everything above.

Each article links back here and to the next, so jump in anywhere.
