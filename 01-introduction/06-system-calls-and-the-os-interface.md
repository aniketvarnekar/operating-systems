# System Calls and the OS Interface

## Learning Objectives

By the end of this section you should be able to:
- Explain what a system call is and how it differs from an ordinary function call
- Explain why user programs cannot simply be trusted to police themselves, and how hardware enforces the boundary instead
- Name the two CPU privilege levels involved and what each one is allowed to do

## Prerequisites

- Topic 1 (What Is an Operating System?)
- Topic 3 (Virtualization) — helpful context, not required

## Motivation

Topics 1–5 described the OS's *responsibilities* (resource management, virtualization, concurrency, persistence) at a conceptual level. This topic answers the concrete, mechanical question: when a program actually needs the OS to do one of these things — start a new process, read a file, allocate memory — what, precisely, happens? This is the last piece needed before Module 02 can start using terms like "system call" without first defining them.

## Problem Statement

Suppose a user program wants to read a file. It cannot simply reach out and manipulate the disk's hardware registers directly — for at least two reasons:

1. **Safety**: if any program could directly manipulate disk hardware, a buggy or malicious program could corrupt data belonging to a completely different program, or the file system's own structures, with nothing to stop it.
2. **Practicality**: raw disk hardware doesn't understand "files" at all — it only understands physical sector addresses. Something has to translate "read this file" into the actual sequence of hardware operations that accomplishes it.

So the program needs a way to say "OS, please do this privileged operation for me" — and critically, that request needs to be something the *hardware itself* can enforce as the only valid path, not merely something the OS politely offers and hopes programs use.

## Concept

### Two Privilege Levels

Modern CPUs support (at least) two execution modes, enforced directly in hardware:

- **User mode**: the mode ordinary programs run in. Certain instructions (directly accessing disk hardware, directly changing which program the CPU is running, directly rewriting memory-protection settings) are simply *forbidden* in this mode — the CPU itself refuses to execute them and raises an exception if attempted.
- **Kernel mode** (also called supervisor/privileged mode): the mode the OS itself runs in. Every instruction, including the ones forbidden in user mode, is permitted here.

This distinction is not a software convention the OS is merely trusting programs to respect — it is physically enforced by the CPU's hardware. A user-mode program *cannot* execute a privileged instruction no matter what it tries; the hardware simply won't allow it. This is precisely the enforcement mechanism that makes the "trust" problem from Topic 1 solvable.

### The System Call

> A **system call** is a special, controlled instruction a user-mode program executes specifically to ask the kernel to perform a privileged operation on its behalf, safely transitioning the CPU from user mode into kernel mode for the duration of that request.

Mechanically, a system call is not an ordinary function call (like calling a function within your own program). It's a deliberate **trap**: a special instruction that:
1. Saves enough of the calling program's state to resume it correctly afterward.
2. Switches the CPU from user mode into kernel mode.
3. Jumps to a specific, predetermined location inside the OS kernel — never to an arbitrary, attacker-chosen address — that knows how to safely handle that specific kind of request.
4. Once the kernel finishes handling the request, switches the CPU back to user mode and resumes the calling program.

The "jump only to a predetermined location" detail matters enormously for safety: if a user program could make the CPU jump into kernel mode at *any* address of its choosing, it could potentially trick the kernel into executing arbitrary, dangerous code with full privilege. Restricting entry to a small, fixed set of kernel-defined entry points closes this off.

## Internal Working (Preview)

```
 User Mode                          Kernel Mode
 ──────────                         ───────────
 program calls read(fd, buf, n)
        │
        ▼
 special trap instruction ───────►  CPU switches to kernel mode,
                                     jumps to the OS's fixed,
                                     predetermined system-call
                                     entry point (never an
                                     arbitrary address)
                                            │
                                            ▼
                                     kernel validates the request,
                                     performs the privileged
                                     operation (e.g., actually
                                     reads from disk hardware)
                                            │
                                            ▼
 program resumes ◄────────────────  CPU switches back to
 with the result                    user mode, returns result
```

## Real-World Analogy

Think of a bank vault. A regular customer (a user-mode program) cannot walk into the vault directly — the vault door is physically locked to anyone without special authorization. Instead, the customer fills out a withdrawal slip and hands it through a teller window (the system call) to a bank employee who *does* have vault access (the kernel, running in kernel mode). The employee validates the request, retrieves exactly what was asked for from the vault, and hands it back through the same window. Crucially, the customer never gets to walk into the vault themselves, even briefly, no matter how they phrase the request — the only path to vault contents is through the teller window, and only the employee ever crosses that boundary.

## Why This Design Is Necessary

The alternative to hardware-enforced privilege levels is a purely software-based "honor system," where the OS simply asks programs nicely not to touch hardware directly — and Topic 1 already established why that fails the instant any single program (accidentally or maliciously) doesn't cooperate. Hardware-enforced user/kernel mode is what makes the OS's isolation and resource-management promises *actually* trustworthy, rather than merely conventional.

## Advantages of This Design

- **Real enforcement, not just convention** — the hardware itself refuses forbidden operations in user mode; no program can simply choose to bypass it.
- **A single, well-defined choke point** — every privileged operation goes through the same kind of controlled trap, giving the OS one consistent place to validate, log, or restrict requests.
- **Safety of the entry point itself** — restricting kernel entry to fixed, predetermined locations prevents user code from jumping into arbitrary, dangerous kernel code paths.

## Disadvantages / Costs

- **Overhead per call** — every system call incurs the real cost of trapping into the kernel and back, which is measurably slower than an ordinary same-mode function call; programs that make excessive numbers of small system calls pay for this repeatedly.
- **API surface to maintain** — the specific set of system calls an OS supports becomes a long-term, hard-to-change contract that all existing software depends on.

## Best Practices

- When reasoning about whether an operation needs OS involvement, ask: "does this touch hardware, another process's resources, or something the OS must arbitrate fairness/safety over?" If yes, it requires a system call; if it's purely local computation within your own program's already-allocated memory, it doesn't.
- Batch small operations where possible (e.g., write a large buffer once instead of one byte at a time) to amortize the real per-call overhead of trapping into the kernel.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A system call is just a regular function call to some OS library code." | A regular function call stays in the same privilege mode. A system call deliberately switches the CPU's privilege mode itself (user → kernel → user), which is a hardware-level transition, not merely a jump to a different address. |
| "User mode is a software restriction the OS could remove if it wanted to be faster." | It's enforced by the CPU hardware itself; removing it would mean any user program could execute arbitrary privileged instructions directly, eliminating isolation and safety entirely. |

## Interview Questions

1. **Q: What is a system call, mechanically?**
   A: A deliberate trap instruction that a user-mode program executes to request a privileged operation; it saves the caller's state, switches the CPU into kernel mode, jumps to a fixed, predetermined OS entry point, performs the request, then switches back to user mode and resumes the caller.

2. **Q: Why can't a user-mode program just directly execute the hardware instructions a system call would trigger?**
   A: Because certain instructions are hardware-restricted to kernel mode only — the CPU itself refuses to execute them in user mode, which is precisely what makes the OS's safety and resource-management guarantees real and unavoidable, not just conventional.

3. **Q: Why must a trap only be able to jump to a small, predetermined set of kernel entry points, rather than any address?**
   A: If user code could direct the CPU to jump into kernel mode at an arbitrary, attacker-chosen address, it could potentially execute unintended, dangerous kernel code with full privilege — restricting entry points closes off this entire class of attack.

## Summary

- The CPU hardware enforces (at least) two privilege levels: user mode (restricted) and kernel mode (full access), which is what makes the OS's isolation promises real rather than conventional.
- A system call is a controlled trap that switches a program from user mode into kernel mode, at a fixed, predetermined entry point, so the OS can safely perform a privileged operation on the program's behalf.
- This mechanism is the concrete "how" behind every abstraction introduced conceptually in Topics 1–5 — every process creation, memory allocation, and file read you'll see starting in Module 02 ultimately goes through this same trap-based boundary.
