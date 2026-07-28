# Article

Technical reference notes. Start with the [intro](intro.md), or jump to any part:

- **[Introduction](intro.md)** — the layers mental model and a reading order for the series.
- **[Machine code](machine-code.md)** — how source becomes machine code (compile + link), static vs dynamic, object vs executable.
- **[The operating system](operating-system.md)** — what runs your code: processes, virtual memory, page protection, and syscalls.
- **[Executable & package formats](executable-formats.md)** — EXE / ELF / APK / IPA as wrappers around native code.
- **[Shellcode](shellcode.md)** — machine code packaged for injection: position-independent, null-free, self-sufficient.
- **[Position-independent code](position-independent-code.md)** — what PIC really means, and why whole-program PIC is hard.
- **[The Position-Independent Agent](position-independent-agent.md)** — a real project that compiles a whole app into pure PIC shellcode.
- **[Appendix A — PowerShell injector](appendix-powershell-injector.md)** — standard W^X shellcode loader in pure PowerShell.
- **[Appendix B — Python injector](appendix-python-injector.md)** — cross-platform shellcode loader (remote-inject on Windows, `mmap` on POSIX).
- **[Appendix C — NoRWX](appendix-norwx.md)** — running shellcode without RWX memory, via fault-driven emulation.
