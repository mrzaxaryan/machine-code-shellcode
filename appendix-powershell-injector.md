# Appendix A — PowerShell Shellcode Injector

> **Appendix to:** [shellcode](shellcode.md) and the [Position-Independent Agent](position-independent-agent.md) · **Back to:** [intro](intro.md)

## What it is
[**nostdlib/PowerShellShellcodeInjector**](https://github.com/nostdlib/PowerShellShellcodeInjector) is a Windows shellcode loader written in pure PowerShell (with inline C# P/Invoke — no external modules, no admin rights). It downloads position-independent code from GitHub Releases and runs it **inside the PowerShell process itself**, with proper **W^X** memory discipline.

## The load sequence (W^X, not RWX)
This is the disciplined version of the standard loader — it never holds a page both writable *and* executable at the same time:

```
   1. download the arch-specific shellcode (.bin) from a GitHub Release
   2. VirtualAlloc        → PAGE_READWRITE     (RW  — write only)
   3. copy the shellcode in
   4. VirtualProtect      → PAGE_EXECUTE_READ  (RX  — execute only)
   5. call it as a function pointer (Func<int> delegate)   →  runs in-process
   6. VirtualFree         → release when done
```

Two things worth noting:

- **W^X, not RWX.** Compare the naïve loader (`VirtualAlloc(RWX)` — write and execute at once, very loud). This injector splits the two: write under RW, then flip to RX before executing. It still ends with an executable page — which is exactly what [Appendix C — NoRWX](appendix-norwx.md) refuses to allocate at all.
- **No `CreateThread`, no child process.** Instead of spawning a thread (a common tell), it invokes the shellcode entry point directly as a **.NET function pointer** (`Func<int>`), so the payload runs on the PowerShell process's own thread.

## Fetching the payload
It auto-detects the CPU via `PROCESSOR_ARCHITECTURE` and pulls the matching asset — `windows-x86_64.bin`, `windows-i386.bin`, `windows-aarch64.bin`, or `windows-armv7a.bin` — then sanity-checks it (rejects too-small payloads and HTML "error page" responses). SSL verification is intentionally disabled, since it's fetching unsigned public blobs.

> **Why it matters.** A clean, readable reference for the *standard* shellcode-loader pattern, and the practical runner for a payload like the [Position-Independent Agent](position-independent-agent.md), whose flat PIC blob is exactly what a loader like this is built to run. For the opposite philosophy (no executable allocation at all), see [Appendix C — NoRWX](appendix-norwx.md) and the cross-platform [Appendix B](appendix-python-injector.md).
