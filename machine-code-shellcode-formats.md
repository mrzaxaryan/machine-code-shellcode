# Machine Code, Shellcode, and Executable Formats (EXE / ELF / APK / IPA)

A running program is built in **layers**. This article walks down those layers:

1. **Machine code** — the raw instructions a CPU actually executes, and how source code becomes it (compiling + linking).
2. **Shellcode** — machine code packaged for *injection* rather than for an OS loader (position-independent, flat bytes).
3. **Executable & package formats** — how that machine code gets wrapped so an operating system can load and run it: **EXE**, **ELF**, **APK**, **IPA**.

The single mental model to keep in mind:

> Machine code is the **payload**. Shellcode, EXE, ELF, APK, and IPA are different **wrappers/containers** for that payload, each meant for a different *loader* (a CPU jump, the Windows PE loader, the Linux ELF loader, Android ART, iOS dyld).

---

## 1. Machine Code

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

## 2. Shellcode

### What it is
**Shellcode** is a compact blob of machine code meant to be **injected into a target process and executed there** — not loaded by an OS as a normal file. Classically it was the payload that gave you a shell (hence the name); today it's any small native payload (reverse shell, meterpreter stager, egg hunter, etc.).

### What makes it shellcode, not just "machine code"
Shellcode is machine code with three extra constraints:

1. **Position-Independent Code (PIC)** — it must run correctly *no matter where in memory it lands*, because the attacker controls where it's copied. So it uses:
   - **PC/RIP-relative addressing** and **relative branches** (`call rel32`, `jmp rel32`), never absolute addresses baked into the bytes.
   - **"GetPC" tricks** (e.g. `call next; next: pop rdi`) to learn its own address at runtime when it needs to reference its own data.
   - It does **not** rely on a loader to fix up addresses — there is no loader.

2. **Bad-character free** — most injection paths copy bytes as a string (`strcpy`, `read` into a buffer), so a `\x00` (null) terminates the copy. Shellcode is hand-written/re-encoded to contain **no null bytes** (and often avoids `\n`, `\r`, spaces, etc., depending on the channel).

3. **Small and self-contained** — it can't assume libc is set up; it reaches the kernel directly through **syscalls** (e.g. `syscall` on x86-64, `svc 0` on ARM).

### Concrete example — Linux x86-64 "exit(0)"

Naïve (has null bytes — fine for an executable, **bad** for shellcode):
```
b8 3c 00 00 00   mov  eax, 60      ; sys_exit  (note the 00 00 00)
bf 00 00 00 00   mov  edi, 0       ; status 0
0f 05            syscall
```

Null-free shellcode version:
```
31 c0   xor  eax, eax     ; eax = 0  (clears upper bits, no nulls)
31 ff   xor  edi, edi     ; edi = 0
b0 3c   mov  al, 60       ; eax low byte = 60  → sys_exit
0f 05   syscall
```
Same result (`exit(0)`), but every byte is non-zero and it has no absolute address — it works wherever it's placed. That is the essence of shellcode.

### "Linked" shellcode
You asked about linking in the shellcode context. Shellcode is usually **hand-assembled** (e.g. `nasm -f bin`) so it emits raw bytes directly — no object headers, no linker. But you *can* also:

- **Compile from C with freestanding flags** (`-nostdlib -ffreestanding -fno-pic` off, PIC on, `-O2`), then use a **custom linker script** to lay everything out contiguously, and finally `objcopy -O binary` to strip headers and dump the flat `.text` as raw bytes. That flat blob is your shellcode.
- **Staged payloads**: a tiny **stage-0** loader (a few dozen bytes) does just enough — `mmap`/`VirtualAlloc` memory, mark it executable, then read or decode the larger **stage-1** payload and jump to it. This is the shellcode analog of "linking": the stage-0 stub binds the bigger payload into place at runtime.

The hard contrast: a normal executable *describes itself* to the OS (headers, sections, imports) and lets the OS place it; **shellcode places and describes nothing — it just runs where it lands.**

---

## 3. Executable & Package Formats: EXE, ELF, APK, IPA

Here is the crucial two-level distinction that makes the comparison click:

- **Executable file formats** (PE/EXE, ELF, Mach-O) — single files with **headers + sections + machine code** that a **native OS loader** parses to run the bytes directly on the CPU.
- **Application package formats** (APK, IPA — and Windows MSIX, Linux deb/rpm) — **ZIP archives** that *bundle* an executable (or bytecode) together with resources, metadata, and a signature, for an app store / mobile OS.

EXE and ELF are level 1. APK and IPA are level 2 (they *contain* level-1 files inside).

### EXE — Windows (Portable Executable, PE)
- **OS:** Windows. The format is **PE/COFF**; `.exe` is a PE whose subsystem is a Windows app, while `.dll` is the same PE format flagged as a library.
- **Contents:** DOS/PE headers, sections (`.text` code, `.data`, `.rdata`, `.rsrc` resources…), an **Import Address Table (IAT)** for DLL dependencies, an **export table**, and optional resources/manifests.
- **Loader:** the Windows image loader maps sections, applies **ASLR** relocations, resolves the IAT against `kernel32.dll`/`ntdll.dll` and friends, then jumps to the entry point.
- **Native machine code?** Yes — x86 / x64 / ARM64 bytes directly.

### ELF — Linux / Unix / BSD (Executable and Linkable Format)
- **OS:** Linux, most Unix-likes, many embedded RTOSes. One format, many uses: **executables**, **shared objects (`.so`)**, **relocatable objects (`.o`)**, and even **core dumps**.
- **Contents:** ELF header, **program headers** (what to map into memory at runtime) and **section headers** (used at link time), plus dynamic tables, the **PLT/GOT** for dynamic linking, and relocations.
- **Loader:** the Linux kernel maps the program-header segments, then hands off to the **dynamic linker `ld.so`**, which resolves shared-library dependencies and patches the GOT/PLT before control reaches `main`.
- **Native machine code?** Yes — directly.

> Note on Apple: macOS / iOS use **Mach-O** (not PE or ELF). This matters because it is the *native* executable format **inside** an IPA — see below.

### APK — Android application package
- **OS:** Android. **It is a ZIP archive, not a native executable format.**
- **Contents (unzipped):**
  - `classes.dex` / `*.dex` — app bytecode in **DEX** format (compiled from Java/Kotlin), later AOT/JIT-compiled to native by **ART** (`dex2oat`).
  - `lib/<abi>/*.so` — native libraries, which are **ELF** files (so ELF *lives inside* the APK).
  - `resources.arsc`, `res/`, `assets/` — compiled resources and raw assets.
  - `AndroidManifest.xml` (compiled binary XML) — the app's manifest.
  - `META-INF/` — the APK signature (v1) and metadata.
- **Runtime:** **ART** (Android Runtime) runs the DEX bytecode; native code in the embedded `.so` ELF libraries runs via the standard Android ELF loader.
- **Native machine code?** Indirectly — the app logic is DEX bytecode; only the bundled `.so` libraries are native ELF.

### IPA — iOS application archive
- **OS:** iOS. **Also a ZIP archive.**
- **Contents (unzipped → `Payload/App.app/`):**
  - The actual app binary — a **Mach-O** executable (fat/universal if it supports multiple architectures).
  - `Info.plist` — app metadata.
  - Frameworks and embedded `.dylib`s (also Mach-O).
  - App resources/assets.
  - `_CodeSignature/` — the mandatory Apple **code signature** (an IPA won't install/run without it).
- **Runtime:** the **`dyld`** dynamic linker maps the Mach-O binary and resolves its dylibs; iOS enforces the code signature and sandbox before launch.
- **Native machine code?** Yes — iOS apps are native Mach-O (compiled ahead of time; no bytecode VM like ART). The IPA is just the *bundle* around that Mach-O binary.

### Side-by-side comparison

| | **EXE (PE)** | **ELF** | **APK** | **IPA** |
|---|---|---|---|---|
| OS | Windows | Linux/Unix/BSD | Android | iOS |
| Level | Executable format | Executable format | App package (ZIP) | App package (ZIP) |
| What it really is | Header + sections | Header + segments/sections | ZIP of DEX + resources + native libs | ZIP of Mach-O binary + resources |
| Native machine code inside? | Directly | Directly | Only the embedded `.so` (ELF) | Yes — the Mach-O binary |
| App logic runs as | Native code | Native code | DEX bytecode on **ART** (+ native libs) | Native Mach-O code |
| Loader / runtime | Windows PE loader + IAT | Kernel + `ld.so` (PLT/GOT) | ART + Android native loader | `dyld` + code-signature check |
| Dependency sharing | `.dll` (dynamic) | `.so` (dynamic) | bundled `.so` / Play delivery | bundled `.dylib` / frameworks |

---

## Putting it all together

- **Machine code** is the universal payload — architecture-specific bytes the CPU runs. *Compiling* turns source into relocatable object code; *linking* resolves it into a runnable executable.
- **Shellcode** is the *same* machine code, but repackaged for injection: position-independent, null-free, tiny, and self-sufficient — it relies on raw syscalls, not a loader.
- **EXE / ELF / Mach-O** are the *native* containers the OS loader understands — headers + sections around machine code, with dynamic-linking machinery.
- **APK / IPA** are a layer *above* that — ZIP packages that bundle a native executable (ELF inside APK, Mach-O inside IPA) with bytecode, resources, and a signature for a mobile OS / app store.

One payload, four increasingly "packaged" forms: **raw bytes → shellcode → OS executable → app store bundle.**
