# Calling Conventions (the ABI)

> **Series:** [assembly](03-assembly-and-encoding.md) ← **calling conventions** → [operating system](05-operating-system.md)

### What it is
Machine code can call functions — but only if caller and callee **agree** on the rules: which register holds the first argument, where the return value goes, who saves which registers, how the stack is laid out. That contract is the **ABI** (Application Binary Interface), and its function-call part is the **calling convention**. Without it, separately-compiled code (your program and the OS's libraries) couldn't talk.

### The big three

| | **System V AMD64** (Linux, macOS) | **Microsoft x64** (Windows) | **AAPCS64** (ARM64) |
|---|---|---|---|
| Integer args | `rdi, rsi, rdx, rcx, r8, r9`, then stack | `rcx, rdx, r8, r9`, then stack | `x0–x7`, then stack |
| Return value | `rax` | `rax` | `x0` |
| Stack at call | 16-byte aligned | 16-byte aligned | 16-byte aligned |
| Caller-saved (volatile) | `rax, rcx, rdx, rsi, rdi, r8–r11` | `rax, rcx, rdx, r8–r11` | `x0–x18` (mostly) |
| Callee-saved (preserved) | `rbx, rbp, r12–r15` | `rbx, rbp, rdi, rsi, r12–r15` | `x19–x29` |
| Extra | 128-byte *red zone* below `rsp` | 32-byte *shadow space* for the callee | link register `x30` |

### Why these details bite
- **Shadow space (Windows).** The caller must reserve 32 bytes on the stack for the callee to spill `rcx–r9`. Skip it and you corrupt the stack.
- **16-byte alignment.** On both x64 conventions `rsp` must be 16-byte aligned at the moment of a `call` — many Windows APIs **fault** on misalignment (they use aligned SSE moves internally). Shellcode that calls Win32 APIs has to set this up by hand.
- **Red zone (SysV).** Leaf functions may use 128 bytes below `rsp` without moving the pointer — handy, but a signal or interrupt can clobber it.

> **Analogy — passing a baton.** The convention is the rule for *which hand* you hold the baton (args) in, *where* you set it down (return value), and *who* is responsible for catching it (saved registers). Change the rule mid-relay and the baton hits the floor.

### Why shellcode & freestanding code care
Shellcode can't call `printf` through libc — it calls the OS, or hand-resolved APIs, directly (covered later, in *runtime API resolution*). To do that it must place arguments **exactly** per the target's convention and keep the stack aligned. The [Position-Independent Agent](19-position-independent-agent.md) does precisely this: when its code calls a Windows API, it forwards the call using the Microsoft x64 fastcall convention.

> **Why it matters.** The ABI is the difference between "these bytes call `WriteFile` correctly" and "these bytes crash." It's also why a blob compiled for Windows won't run on Linux unchanged — same CPU, different convention.

---

▶ **Next:** The environment that enforces all of this and provides the APIs you call — **[the operating system](05-operating-system.md)**.
