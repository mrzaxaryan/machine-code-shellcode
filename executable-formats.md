# Executable & Package Formats: EXE, ELF, APK, IPA

> **Series:** [machine code](machine-code.md) ← **formats** → [shellcode](shellcode.md)

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

▶ **Next:** The flip side — machine code packaged for *injection* rather than an OS loader: **[shellcode](shellcode.md)**.
