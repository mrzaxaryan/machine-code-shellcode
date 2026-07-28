# The Operating System

> **Series:** [machine code](machine-code.md) ← **operating system** → [executable formats](executable-formats.md)

### What it is
Machine code is the payload — but on its own, a raw blob of bytes can't do anything useful. It can't print to a screen, open a file, or get memory without help. The **operating system (OS)** is the always-running layer between your code and the hardware that makes all of that possible: it runs your program, gives it memory, and mediates every access to the CPU, disk, and devices.

Three jobs to keep in mind:

- **Run programs** — turn an executable file into a live **process** with its own memory and a thread of execution.
- **Manage memory** — hand each process a private **virtual address space**, in pages tagged with read/write/execute permissions.
- **Gate the hardware** — your code runs unprivileged; the only way to touch hardware (disk, network, screen) is to *ask the OS* through a controlled **syscall**.

### Processes: how an executable comes alive
This is the moment everything in the series actually starts running. The OS **loader** (the Windows PE loader, Linux's kernel + `ld.so`, Apple's `dyld`) does roughly:

```
   exec ./hello
      │
      ▼
   1. read the executable's headers
   2. create a new process + virtual address space
   3. map code/data segments into memory (with R/W/X permissions)
   4. resolve dynamic deps (PLT/GOT, IAT) and run CRT startup
   5. jump to the entry point   →   your code runs
```

A "running program" is really just: an address space full of mapped pages plus at least one thread executing machine code inside it. (The format-specific details of steps 1–4 are exactly what [executable formats](executable-formats.md) covers.)

> **Analogy — a hotel.** The OS is the hotel: it gives each guest (process) their own room (address space) and a keycard that opens only their door. Guests can't wander into the boiler room (hardware) themselves — they call the front desk (a syscall) and ask.

### Memory & protection: pages and permissions
The OS doesn't hand out raw RAM. It gives each process a **virtual address space** divided into fixed-size **pages** (typically 4 KB), and every page carries **permission bits**: read, write, execute. Two consequences that matter a lot for this series:

- **Isolation** — process A can't read or write process B's memory; each sees only its own virtual world.
- **W^X / DEP-NX** — the hardware (with the OS) refuses to *execute* code from a page that isn't marked executable. This is the exact protection the [NoRWX appendix](appendix-norwx.md) routes around, and the reason shellcode loaders must allocate executable (RWX/RX) pages.

### User mode vs kernel mode
CPUs run at privilege **levels** (rings). Your program runs in **user mode** (unprivileged); the OS kernel runs in **kernel mode** (full hardware access). A user-mode program literally *cannot* execute the privileged instructions that touch hardware or page tables — there's no opcode it can run to bypass that. The only door between them is the **syscall**.

```
   user mode  ──syscall (write, open, mmap, …)──▶  kernel mode
   (your code,                                 │   (the OS: checks args/permissions,
    unprivileged)                              │    does the real work, touches hardware)
                ◀─────────── result ------------┘
```

This is why "reach the kernel directly through syscalls" (see [shellcode](shellcode.md) and [PIA](position-independent-agent.md)) isn't a stylistic choice — syscalls are the *only* legal way for user-mode code to get anything done.

### Syscalls: the controlled gateway
A **syscall** is a documented, numbered request: "kernel, please do operation N with these arguments." On x86-64 Linux that's the `syscall` instruction with a number in `rax` (e.g. `rax = 1` → `write`); on Windows you call into `ntdll.dll`, which issues the syscall for you. The kernel checks the arguments and permissions, does the work, and returns a result. Shellcode and freestanding programs ([PIA](position-independent-agent.md)) skip libc and issue these directly — libc's `printf`/`open` are just wrappers around the very same syscalls.

> **Why it matters.** Every wrapper in this series exists to be loaded and run by an OS, and every "reach the kernel directly" trick is just using the OS's own syscall gateway. The OS is the stage; machine code, shellcode, EXE/ELF/APK/IPA are the actors and costumes.

---

▶ **Next:** How machine code gets packaged so this loader can run it — **[executable & package formats](executable-formats.md)** (EXE / ELF / APK / IPA).
