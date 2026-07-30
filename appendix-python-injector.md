# Appendix B — Python Shellcode Injector

> **Appendix to:** [shellcode](09-shellcode.md) and the [Position-Independent Agent](19-position-independent-agent.md) · **Back to:** [intro](01-introduction.md)

## What it is
[**nostdlib/PythonShellcodeInjector**](https://github.com/nostdlib/PythonShellcodeInjector) is a **cross-platform** shellcode loader in pure Python (stdlib only: `ctypes`, `mmap`, `urllib`; runs on Python 2.6+ and 3.x). It downloads position-independent code matching the host OS/arch and runs it — using **different techniques per platform**.

## Windows: remote-process injection
Unlike the [PowerShell injector](appendix-powershell-injector.md) (which runs in-process), on Windows this one injects into a fresh host process — the classic suspended-process technique:

```
   1. CreateProcessW       → start a SUSPENDED cmd.exe   (the host process)
   2. VirtualAllocEx       → PAGE_READWRITE in it        (RW)
   3. WriteProcessMemory   → copy the shellcode in
   4. VirtualProtectEx     → PAGE_EXECUTE_READ           (RX)
   5. CreateRemoteThread   → run the shellcode in the host
   6. resume / wait
```

Because it targets a *native-arch* host process, "any Python bitness works" — 32-bit Python can inject into a 64-bit `cmd.exe` when that's the host arch.

## POSIX (Linux / macOS / BSD / Solaris / Android / iOS): in-process
On Unix-likes it stays in-process and uses the POSIX equivalents:

```
   1. mmap     → PROT_READ | PROT_WRITE   (RW)
   2. copy the shellcode in
   3. mprotect → PROT_READ | PROT_EXEC    (RX)
   4. call it as a function pointer
```

## Common to both
- **W^X discipline** — RW while writing, flipped to RX before executing; never RWX. (Contrast [Appendix C — NoRWX](appendix-norwx.md), which dispenses with executable pages entirely.)
- **Remote fetch + auto-detect** — downloads `{platform}-{arch}.bin` from a URL template based on the detected OS/arch (including emulated cases like x86-64 Python on ARM64 Windows), with size and HTML-response validation.
- **SSL verification disabled** by design.

> **Why it matters.** It shows the *same* W^X loader idea expressed two ways — remote-process injection on Windows, in-process `mmap`/`mprotect` on POSIX — and is a natural runner for a cross-platform PIC blob like the [Position-Independent Agent](19-position-independent-agent.md). For the stealth extreme (run from non-executable memory, no executable allocation at all), see [Appendix C — NoRWX](appendix-norwx.md).
