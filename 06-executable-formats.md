# Executable & Package Formats: EXE, ELF, APK, IPA

> **Series:** [operating system](05-operating-system.md) ← **formats** → [linking](07-linking-and-relocations.md)

Here is the crucial two-level distinction that makes the comparison click:

- **Executable file formats** (PE/EXE, ELF, Mach-O) — single files with **headers + sections + machine code** that a **native OS loader** parses to run the bytes directly on the CPU.
- **Application package formats** (APK, IPA — and Windows MSIX, Linux deb/rpm) — **ZIP archives** that *bundle* an executable (or bytecode) together with resources, metadata, and a signature, for an app store / mobile OS.

EXE and ELF are level 1. APK and IPA are level 2 (they *contain* level-1 files inside).

> **Analogy — meal vs. meal kit.** An executable format (EXE/ELF/Mach-O) is a *ready-to-eat meal* the OS's microwave (its loader) just heats and serves. A package format (APK/IPA) is a *meal-kit box* — a ZIP — containing ingredients (the native binary + bytecode), a recipe (the manifest/`Info.plist`), and a tamper-evident seal (the signature).

The level-2 packages literally unzip to reveal level-1 files inside:

```
   APK (a ZIP archive)              IPA (a ZIP archive)
   ├── classes.dex      (DEX)       ├── Payload/App.app/
   ├── lib/<abi>/*.so   (ELF) ◀─    │   ├── App          (Mach-O) ◀─
   ├── AndroidManifest.xml          │   ├── Frameworks/*.dylib (Mach-O)
   ├── res/, assets/                │   └── Info.plist
   └── META-INF/        (signature) └── _CodeSignature/   (signature)
```

Notice the native **ELF** `.so` living *inside* the APK, and the native **Mach-O** binary living *inside* the IPA — the package is just a wrapper around a level-1 executable.

### In practice: peeking inside an APK
```
$ unzip -l app.apk                       # it's a ZIP, not an executable
   ... classes.dex, lib/arm64-v8a/libnative.so, res/, AndroidManifest.xml ...
$ file lib/arm64-v8a/libnative.so        # "ELF 64-bit LSB shared object, ARM aarch64"
```

### EXE — Windows (Portable Executable, PE)
- **OS:** Windows. The format is **PE/COFF**; `.exe` is a PE whose subsystem is a Windows app, while `.dll` is the same PE format flagged as a library.
- **Contents:** DOS/PE headers, sections (`.text` code, `.data`, `.rdata`, `.rsrc` resources…), an **Import Address Table (IAT)** for DLL dependencies, an **export table**, and optional resources/manifests.
- **Loader:** the Windows image loader maps sections, applies **ASLR** relocations, resolves the IAT against `kernel32.dll`/`ntdll.dll` and friends, then jumps to the entry point.
- **Native machine code?** Yes — x86 / x64 / ARM64 bytes directly.

#### File structure (top to bottom)
```
   ┌─ DOS header        (starts with "MZ"; e_lfanew → offset of the PE header)
   ├─ DOS stub          (legacy 16-bit stub)
   ├─ PE signature      ("PE" + 2 NUL bytes)
   ├─ COFF file header  (machine, #sections, timestamp, characteristics)
   ├─ Optional header   (PE32 / PE32+: ImageBase, AddressOfEntryPoint,
   │                    SectionAlignment, SizeOfImage, + a data-directory table)
   ├─ Section table     (per section: name, VirtualAddress, VirtualSize,
   │                    raw offset/size, characteristics)
   └─ Sections:  .text (code)  .rdata (read-only)  .data  .rsrc (resources)  .reloc
```
Key fields the loader cares about: `AddressOfEntryPoint` (the RVA it jumps to after setup), `ImageBase` (preferred load address — **ASLR** rebases from here), and the **data directories** — tables it walks for imports (→ **IAT**), exports, base relocations, and resources.

### ELF — Linux / Unix / BSD (Executable and Linkable Format)
- **OS:** Linux, most Unix-likes, many embedded RTOSes. One format, many uses: **executables**, **shared objects (`.so`)**, **relocatable objects (`.o`)**, and even **core dumps**.
- **Contents:** ELF header, **program headers** (what to map into memory at runtime) and **section headers** (used at link time), plus dynamic tables, the **PLT/GOT** for dynamic linking, and relocations.
- **Loader:** the Linux kernel maps the program-header segments, then hands off to the **dynamic linker `ld.so`**, which resolves shared-library dependencies and patches the GOT/PLT before control reaches `main`.
- **Native machine code?** Yes — directly.

#### File structure (top to bottom)
```
   ┌─ ELF header (Elf64_Ehdr)
   │     magic bytes 7F 45 4C 46 ("ELF"), class (32/64), data (endian),
   │     type (ET_EXEC / ET_DYN), machine, e_entry, e_phoff, e_shoff, e_shstrndx
   ├─ Program headers (Elf64_Phdr)   ◀── runtime: what to map
   │     PT_LOAD (segments), PT_INTERP (the ld.so path), PT_DYNAMIC, PT_GNU_STACK (NX)
   ├─ Sections: .text .rodata .data .bss .got .plt .dynamic .symtab .strtab
   └─ Section headers (Elf64_Shdr)   ◀── link time: name, offset, size, type
```
ELF uniquely keeps **both** views: **program headers** describe the *segments* the loader maps (runtime), while **section headers** describe *sections* for linking and tooling (optional at runtime — stripped binaries drop them). `e_entry` is where execution begins.

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

#### File structure (it's a ZIP)
```
   APK = a ZIP archive
   ┌─ Local file headers + data:
   │     AndroidManifest.xml           (compiled binary XML — the "recipe")
   │     classes.dex  [classes2.dex …] (DEX bytecode)
   │     resources.arsc                (compiled resource table)
   │     res/   assets/                (resources & raw assets)
   │     lib/<abi>/lib*.so             (native libs — ELF ◀── a level-1 file inside)
   │     META-INF/{MANIFEST.MF, *.SF, *.RSA}   (v1 signature, per-file)
   └─ End Of Central Directory record          (the ZIP index; locates every entry)
        (+ APK Signature Scheme v2/v3: a block appended between the last entry
         and the EOCD — signs the *whole* archive, not individual files)
```
Because it's a standard ZIP, `unzip` reads it directly. The Android-specific parts are the compiled `AndroidManifest.xml`, the `.dex` files, `resources.arsc`, and the signature schemes.

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

▶ **Next:** How symbols get resolved and the binary is finalized — **[linking & relocations](07-linking-and-relocations.md)**.
