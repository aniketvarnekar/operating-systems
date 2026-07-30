# Restricted Operations and Traps

## Learning Objectives

By the end of this section you should be able to:
- Explain what a restricted operation is and give concrete examples
- Explain what a trap table is, when it's set up, and why it must be set up in kernel mode
- Trace, step by step, what happens when a process attempts a restricted operation

## Prerequisites

- Topic 1 (The Limited Direct Execution Protocol)
- Module 01, Topic 6 (System Calls and the OS Interface)

## Motivation

Module 01, Topic 6 introduced system calls and the user/kernel mode boundary conceptually. This topic fills in the missing mechanical detail: exactly *how* does the hardware know where to jump to inside the kernel when a trap occurs, and why can't a user process simply choose to jump into kernel code at an address of its own choosing?

## Problem Statement

Suppose a user-mode process executes an instruction that requests a privileged operation — say, a system call asking to read a file. The CPU needs to switch into kernel mode and start executing the *correct* kernel code to handle that specific request. But where, in memory, is that code located? If the process itself could specify the exact memory address to jump to, it could potentially specify the address of some other, dangerous piece of kernel code — or a fabricated address of its own choosing — and trick the kernel into doing something unintended while running with full privilege. How does the hardware guarantee that the *only* code a process can ever enter kernel mode into is code the kernel itself explicitly designated in advance?

## Concept

### Restricted Operations

> A **restricted operation** is any CPU instruction that only kernel mode is permitted to execute — attempting it in user mode causes the hardware to immediately fault instead. Common examples include: directly issuing I/O commands to hardware devices, modifying memory-protection settings, and changing which privilege mode the CPU is currently in.

These operations are restricted precisely because they're the ones capable of breaking isolation or fairness between processes if any process could invoke them freely (Module 01, Topic 1) — reading another process's disk data, corrupting memory-protection boundaries, or seizing permanent control of the CPU.

### The Trap Table

The solution to "where does the hardware jump to on a trap" is set up once, at boot time, while the machine is still fully trusted and running only OS code:

> At boot, the OS (running in kernel mode, before any user process exists) registers a **trap table** with the hardware — a list of specific memory addresses, each one the entry point for handling one particular kind of trap (a specific system call, a divide-by-zero error, a timer interrupt, and so on). Once this table is registered, the hardware itself remembers it, and will only ever jump to one of these pre-registered addresses when a trap occurs — never anywhere else, and critically, a user-mode process has **no way to modify this table or override where the hardware jumps**, since doing so is itself a restricted operation.

This is the exact detail that closes off the danger from the Problem Statement: a user process can *trigger* a trap (by executing a special trap instruction, e.g., as part of a system call), but it never gets to *choose* where that trap lands — the hardware always consults the trap table the kernel itself set up in advance, at a moment when only trusted kernel code was running.

### Trace: What Happens on a Restricted-Operation Trap

1. A user process executes a trap instruction (e.g., as part of calling a library function like `read()`, which itself issues the actual trap instruction under the hood).
2. The trap instruction typically also specifies a **system call number**, identifying exactly which service is being requested (e.g., "this is a request to read a file," as opposed to "write a file" or "create a process").
3. The hardware saves enough of the process's state to resume it later (this overlaps with the context switch, Topic 4), switches to kernel mode, and jumps to the trap table's registered handler for that system call number.
4. The kernel handler validates the request (e.g., checks the process actually has permission to read that specific file) and performs the actual privileged operation.
5. The kernel executes a special "return-from-trap" instruction, which switches the CPU back to user mode and resumes the process — either right after its original trap instruction, or, if the OS decided to switch to a different process instead (Topic 3, Topic 4), by resuming a different process entirely.

## Internal Working (Preview)

```
 BOOT TIME (kernel mode only, fully trusted):
   OS registers trap table:
     entry 0  → syscall handler for read()
     entry 1  → syscall handler for write()
     entry 2  → syscall handler for fork()
     ...
     entry N  → divide-by-zero handler
     entry N+1 → timer interrupt handler (Topic 3)
   Hardware remembers this table. It is now
   permanently protected from user-mode modification.

 RUN TIME (a process is executing):
   user process executes: trap(SYSCALL_READ)
          │
          ▼
   hardware saves state, switches to kernel mode,
   looks up entry "SYSCALL_READ" in the trap table
   (NOT an address chosen by the user process itself)
          │
          ▼
   kernel's read() handler runs, validates + services request
          │
          ▼
   "return-from-trap" ──► back to user mode, resume process
```

## Real-World Analogy

Think of the trap table like a hospital's pre-approved list of emergency call-button destinations, fixed and posted before any patients are ever admitted. A patient (a user process) can press a call button (execute a trap instruction) to summon help, and can even select *which kind* of help they need from a small, fixed menu (a specific system call number — "I need water," "I need a doctor") — but the patient can never rewire the call button to ring an arbitrary phone number of their own choosing, or dial the hospital's restricted operating-room line directly. The hospital administration (the kernel, at boot) is the only one who ever gets to configure which staff member's phone rings for which button, long before any patient is ever admitted.

## Why This Design Is Necessary

If a trap could jump to *any* address the user process specified, the entire safety guarantee of "user mode is restricted" collapses — a process could simply trap to the address of some sensitive, unprotected piece of kernel code and trick it into misbehaving with full privilege. Restricting trap destinations to a small, kernel-chosen, boot-time-registered table — one that user mode cannot alter — is precisely what makes "the kernel is always in control of what runs with elevated privilege" an actual guarantee rather than a hopeful convention.

## Advantages of This Design

- **A small, well-defined, auditable set of kernel entry points** — every possible way into kernel mode is known and fixed in advance, not discovered dynamically at runtime.
- **User processes cannot escalate privilege in unintended ways** — even a maliciously crafted trap instruction can only ever land at one of the kernel's own pre-chosen, legitimate entry points.

## Disadvantages / Costs

- **Every restricted operation still costs a real trap** — the mode switch, state save, and trap-table lookup are all genuine overhead, part of the broader system-call cost discussed in Module 01, Topic 6.
- **The trap table itself is a single, security-critical structure** — a bug in how the OS sets it up at boot, or in how any individual handler validates its inputs, is a serious vulnerability, since it's the sole gatekeeper for all privileged access.

## Best Practices

- When reasoning about "can a user program cause damage by choosing where to jump," remember the trap table closes this off entirely — the destination is never user-controlled, only the request type (via the system call number) is.
- Always validate inputs rigorously inside kernel-mode trap handlers — since they run with full privilege, any way to trick a *legitimate* handler into misusing its own privilege (rather than jumping somewhere illegitimate) becomes the real remaining attack surface.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A user process can choose the exact kernel address a system call jumps to." | It can only choose *which system call* to request (via a system call number), which the hardware then maps to a fixed, kernel-chosen address via the trap table — never an arbitrary address supplied by the process itself. |
| "The trap table can be updated at any time, including by user programs, to add new syscalls." | It's registered once, at boot, exclusively in kernel mode; modifying it is itself a restricted operation, permanently unavailable to user-mode code. |

## Interview Questions

1. **Q: What is a restricted operation, and why must it be restricted?**
   A: A CPU instruction (like direct hardware I/O or changing privilege mode) that only kernel mode may execute; restricting it prevents any user process from directly breaking isolation, fairness, or safety guarantees the OS is responsible for enforcing.

2. **Q: What is a trap table, and when is it set up?**
   A: A boot-time-registered list mapping each kind of trap (system calls, hardware exceptions, timer interrupts) to a specific, fixed kernel-code entry point. It's set up once at boot, while only trusted kernel code is running, and cannot be modified by user-mode code afterward.

3. **Q: Why can't a user process simply jump directly into kernel code at an address of its own choosing?**
   A: Because the hardware enforces that traps only ever land at addresses listed in the kernel-configured trap table — a user process can request *which* registered service it wants (via a syscall number) but has no mechanism to specify or redirect the actual jump destination itself.

## Summary

- Restricted operations are instructions only kernel mode may execute, closing off the exact dangers unrestricted hardware access would create (Module 01, Topic 1).
- The trap table, registered once at boot in kernel mode, is the hardware-enforced, fixed list of legitimate kernel entry points a trap can ever land at.
- A user process can trigger a trap and specify which service it wants (a syscall number), but never *where* the hardware jumps — that's always determined by the kernel's own pre-registered table.
- The next topic covers the second half of "limited": how the OS guarantees it gets the CPU back even from a process that never voluntarily traps at all.
