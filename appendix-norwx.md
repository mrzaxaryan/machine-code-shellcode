# Appendix C — Running Shellcode Without RWX Memory (NoRWX)

> **Appendix to:** [shellcode](09-shellcode.md) and the [Position-Independent Agent](19-position-independent-agent.md) · **Back to:** [intro](01-introduction.md)

## The problem: shellcode needs somewhere executable to run
Recall the standard shellcode loader (from [shellcode](09-shellcode.md) and [PIA](19-position-independent-agent.md)):

```
   1. VirtualAlloc / mmap  → RWX memory      ◀── allocate executable space
   2. memcpy the shellcode in
   3. jump to it
```

(Worked examples of the standard loader — including the more polished **RW → RX** variant — are in [Appendix A](appendix-powershell-injector.md) and [Appendix B](appendix-python-injector.md). NoRWX is what you get when you refuse the executable page *entirely*.)

That first step is the tell. Modern systems enforce **W^X** ("write XOR execute") and **DEP/NX** — a page is writable *or* executable, never both, and data pages can't be executed at all. Defenders (EDR/AV) lean on exactly this: an allocation of **RWX** memory, or even RW-then-flipped-to-RX, is a loud "someone is about to run injected code" signal. So the real question becomes:

> Can you run shellcode **without ever allocating executable memory yourself?**

[**nostdlib/NoRWX**](https://github.com/nostdlib/NoRWX) says *yes* — a Windows engine that runs position-independent code from pages marked only `PAGE_READWRITE`.

## The trick: let the CPU fault, then emulate
NoRWX never marks anything executable. It uses the *same* protection that is trying to stop it:

```
   1. VirtualAlloc  → PAGE_READWRITE        (NOT executable — just RW)
   2. memcpy the shellcode in
   3. register a Vectored Exception Handler (VEH)
   4. jump into the buffer
          │
          ▼   the CPU faults: access violation (no execute permission)
   5. VEH catches the fault
          │
          ▼   software-emulate each x86/x64 instruction
   6. when the shellcode calls an external API
          │     → run that API natively (real calling convention, full speed)
          │     → resume emulation at the return address
```

So the shellcode bytes sit in *non-executable* memory and are **interpreted by a small in-process x86/x64 emulator** rather than run by the CPU. The only code the CPU actually executes is (a) the emulator itself and (b) whatever real Windows APIs the shellcode calls — both of which live in legitimate, already-executable memory.

> **Analogy — a customs checkpoint.** A normal loader smuggles the payload through in a diplomatic pouch (RWX memory the CPU runs directly). NoRWX walks the payload up to the gate in a *non*-executable box; the box is refused entry (the fault), so a handler opens it and reads the contents aloud, one instruction at a time (emulation) — handing off only the *legal* parcels (API calls) for normal delivery.

## Why it matters
- **No RWX, no RX.** NoRWX allocates nothing executable, defeating detectors that hunt for the classic `VirtualAlloc(RWX)` / RW→RX pattern.
- **The shellcode is unchanged.** Because it is emulated, ordinary PIC shellcode (even a blob like the [Position-Independent Agent](19-position-independent-agent.md)'s) runs as-is — PIC is exactly what lets the emulator place it on the fly.
- **A counterpoint to the standard loader.** The PIA loader is *fast and simple* (`RWX → memcpy → jump`); NoRWX is *stealthy and slow* (emulate every instruction). Same payload, opposite tradeoff.

It targets Windows x86-64 and i386 (a VEH plus a per-ISA instruction decoder/handler set), built with Clang/CMake/Ninja — another single-author reference for how much of the "rules" around native-code execution are really conventions you can route around.
