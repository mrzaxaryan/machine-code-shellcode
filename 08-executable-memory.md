# Executable Memory

> **Series:** [linking](07-linking-and-relocations.md) ← **executable memory** → [shellcode](09-shellcode.md)

### What it is
Code can only run if the bytes sit in memory the CPU is allowed to **execute**. The OS doesn't grant that by default — memory you get for data is readable/writable but not executable. This chapter is about that permission, and the dance you do to get an executable page.

### Pages and permissions
The OS hands each process a **virtual address space** carved into fixed-size **pages** (typically 4 KB). Every page carries **permission bits**: **R**ead, **W**rite, e**X**ecute. A data page is `RW`; a normal code page is `RX`. Two consequences matter for this series:

- **Isolation** — process A can't touch process B's pages; each sees only its own virtual world.
- **W^X / DEP-NX** — a page is never both writable *and* executable at once (the "Write XOR Execute" rule), enforced by the hardware. Data pages can't be executed at all — this is **DEP** (Data Execution Prevention) / **NX**.

### Getting an executable page
To run code you did *not* load through the normal OS loader (i.e. shellcode), you have to obtain an `X` page yourself:

```
   Windows:
     VirtualAlloc(PAGE_EXECUTE_READWRITE)                 ← RWX (lazy), or
     VirtualAlloc(PAGE_READWRITE) → copy → VirtualProtect(PAGE_EXECUTE_READ)   ← W^X

   POSIX:
     mmap(PROT_READ|PROT_WRITE) → copy → mprotect(PROT_READ|PROT_EXEC)         ← W^X
     (or mmap(... | PROT_EXEC) directly)
```

### RWX vs the W^X discipline
The lazy way is **RWX** — one page that's writable *and* executable, so you write the shellcode and jump. But that simultaneous W+X is exactly what defenders hunt for. Disciplined loaders flip permissions instead: write under `RW`, then change to `RX` before executing (W^X). That's what [Appendix A](appendix-powershell-injector.md) and [Appendix B](appendix-python-injector.md) do; [Appendix C — NoRWX](appendix-norwx.md) goes further and never allocates an `X` page at all.

> **Analogy — two rooms.** W^X is the rule that the *welding room* (writable) and the *test-firing room* (executable) are never the same room. RWX is one room where you weld and fire simultaneously — productive, but loud and dangerous.

> **Why it matters.** Every shellcode loader in this series is, at bottom, a recipe for obtaining one of these `X` pages and jumping into it. The permission bits are the whole reason [shellcode](09-shellcode.md) needs a loader at all.

---

▶ **Next:** Machine code packaged to live on just such a page — **[shellcode](09-shellcode.md)**.
