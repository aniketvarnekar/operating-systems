# The Context Switch

## Learning Objectives

By the end of this section you should be able to:
- Define a context switch precisely, and distinguish it from a mode switch
- List, in order, the concrete steps the OS takes to switch from one process to another
- Explain the role of the kernel stack in making this switch possible

## Prerequisites

- Module 02, Topic 3 (The Process Control Block)
- Topic 3 (Timer Interrupts and Regaining Control)

## Motivation

Topics 2 and 3 explained the two ways control returns to the kernel (a trap, or a timer interrupt). This topic answers what happens immediately after: if the OS decides to run a *different* process than the one that was just interrupted, how exactly does it switch the CPU from running one process's instructions to running another's, with both processes fully unaware anything happened when they eventually resume?

## Problem Statement

At the instant a trap or timer interrupt fires, the CPU's registers (Module 02, Topic 1) still hold the *interrupted* process's exact working state — including its program counter, marking exactly where it was about to continue. If the OS wants to run a completely different process next, it must:

1. Not lose the interrupted process's state, so that process can be resumed perfectly later.
2. Load a *different* process's previously-saved state into those same physical registers.
3. Do all of this using kernel code that itself needs to run *somewhere*, using CPU registers, without immediately clobbering the very state it's trying to save.

That third point is subtler than it sounds: the kernel's own switching code needs some place to store its own temporary values while it works — but it can't just use the interrupted process's stack, since that stack belongs to a different, less-trusted context.

## Concept

### Definition

> A **context switch** is the OS operation of saving the complete register state of the currently running process into its PCB (Module 02, Topic 3), and loading a different process's previously-saved register state from its own PCB into the CPU — after which execution resumes as that new process, from exactly where it last left off.

This is distinct from a **mode switch** (switching between user mode and kernel mode, discussed in Topics 2–3), which happens on every single trap or interrupt, even when the OS ultimately decides to resume the *same* process afterward. A context switch — switching to a genuinely *different* process — is a strictly more expensive operation that only happens when the OS's scheduling decision (Modules 04–05) is to run someone else.

### The Kernel Stack

Every process actually has **two** stacks associated with it: its ordinary user-mode stack (used by its own function calls while running in user mode), and a separate **kernel stack** — a small, protected region of memory used exclusively by kernel code while handling a trap or interrupt *on behalf of* that process. The kernel stack exists precisely to answer the "where does the switching code itself keep its own temporary state" problem from the Problem Statement: kernel code, running in kernel mode, uses this separate, trusted stack rather than the interrupted process's own (less trusted, user-mode) stack.

### Step-by-Step: What a Context Switch Actually Does

1. A trap or timer interrupt occurs (Topics 2–3); hardware automatically saves a minimal amount of state (like the program counter) onto the current process's kernel stack, and switches to kernel mode.
2. The kernel's trap/interrupt handler runs, using the kernel stack for its own needs, and saves the *remaining* general-purpose register values of the interrupted process into that process's PCB.
3. The scheduler (Modules 04–05) decides which process should run next — it might be the same process that was just interrupted, or a different one.
4. If it's a **different** process: the kernel loads that other process's previously-saved register values from *its* PCB, switches the current stack pointer to point at that other process's kernel stack, and updates whatever memory-management state distinguishes one process's address space from another (previewed here, covered fully starting Module 06).
5. A "return-from-trap" instruction executes, restoring the (now newly-loaded) process's remaining state, switching back to user mode, and resuming execution exactly at that process's saved program counter.

Critically, from the perspective of *either* process involved, nothing appears to have happened at all — each one simply resumes running its own code from exactly where it left off, with no way to detect that time passed, or that a completely different process ran in between.

## Internal Working (Preview)

```
 Process A running  ──trap/interrupt──►  hardware saves minimal state
                                          onto A's kernel stack, switches
                                          to kernel mode
                                                  │
                                                  ▼
                                          kernel handler saves A's
                                          remaining registers into
                                          A's PCB  (Module 02, Topic 3)
                                                  │
                                                  ▼
                                          scheduler decides: run B next
                                                  │
                                                  ▼
                                          kernel loads B's saved
                                          registers FROM B's PCB,
                                          switches to B's kernel stack
                                                  │
                                                  ▼
                                          "return-from-trap"
                                                  │
                                                  ▼
                                  Process B resumes exactly where
                                  IT last left off — unaware that
                                  any time passed, or that A ever ran
```

## Real-World Analogy

Recall the film-set continuity-notes analogy from Module 02, Topic 3 — the context switch is the actual, physical act of the script supervisor walking over, writing down actor A's exact continuity details the instant filming pauses on their scene, then separately walking to actor B's set and reading *their* previously-written continuity notes aloud so B can resume exactly where they left off. The supervisor's own clipboard and pen (the kernel stack) is a tool *they* use to do this work — it's deliberately separate from either actor's own personal props or belongings (each process's own user-mode stack), because mixing them up would risk corrupting the very continuity information being carefully transferred.

## Why This Design Is Necessary

Without a dedicated, protected kernel stack, the OS's own switching code would have nowhere safe to keep its temporary working state while performing a delicate operation — saving and restoring another context — using the very same CPU registers it's trying to manage. Separating the kernel stack from each process's user stack, and carefully following the same fixed save/decide/load/return sequence every time, is what guarantees a context switch is always fully correct and fully invisible to both processes involved, no matter how many times it happens or in what order.

## Advantages of This Design

- **Complete transparency** — a process can never detect, from its own execution, that it was ever paused and resumed, or how much real time passed while it wasn't running.
- **Correctness under arbitrary interleaving** — because the full state save/restore is always complete and consistent, processes can be switched in and out in any order, any number of times, without ever corrupting either one's execution.

## Disadvantages / Costs

- **Real, measurable time cost** — a context switch involves multiple memory writes/reads (saving and loading full register sets) and, if the CPU has caches or a TLB (Module 09), can incur additional indirect costs as those structures "warm up" again for the newly-scheduled process. This cost is a first-class consideration in scheduling policy design (Modules 04–05): switching too frequently wastes CPU time purely on switching overhead rather than useful work.

## Best Practices

- When someone asks "why not just switch processes as often as possible for maximum fairness," point to context-switch overhead as the direct cost that makes "switch constantly" a bad idea in practice — this exact trade-off drives much of the scheduling policy discussion in Modules 04–05.
- Keep "mode switch" (user↔kernel, happens on every trap/interrupt) and "context switch" (switching to a genuinely different process, happens only when the scheduler decides to) as two distinct, separately-costed events in your mental model.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A context switch happens every single time a trap or interrupt occurs." | A mode switch (user↔kernel) happens on every trap/interrupt. A context switch — the more expensive full save/load of a *different* process's state — only happens when the scheduler decides to run someone other than the process that was just interrupted. |
| "The kernel uses the interrupted process's own stack to do its switching work." | It uses a separate, dedicated kernel stack for that process, specifically to avoid using the same (less trusted, user-mode) stack it's in the middle of carefully saving and protecting. |

## Interview Questions

1. **Q: What is a context switch, precisely?**
   A: Saving the complete register state of the currently running process into its PCB, then loading a different process's previously saved register state from its own PCB into the CPU, so execution resumes as that new process from exactly where it last left off.

2. **Q: What's the difference between a mode switch and a context switch?**
   A: A mode switch is the user-mode-to-kernel-mode transition that happens on every trap or interrupt, even if the OS resumes the same process afterward. A context switch is the additional, more expensive step of switching to a genuinely different process's saved state — it only happens when the scheduler chooses to run someone else.

3. **Q: Why does each process have a separate kernel stack, distinct from its own user-mode stack?**
   A: Because kernel code handling a trap or interrupt on that process's behalf needs its own safe, trusted place to store temporary state while it works — using the process's own (less trusted) user-mode stack for this would risk corrupting the very state the kernel is trying to carefully save and restore.

## Summary

- A context switch saves the current process's full register state into its PCB and loads a different process's saved state from its own PCB, resuming it exactly where it left off.
- It's distinct from (and strictly more expensive than) a mode switch, which happens on every trap/interrupt regardless of whether the OS ultimately resumes the same process.
- Each process has a dedicated kernel stack, separate from its user-mode stack, giving kernel code a safe place to work while performing the switch.
- This module has now fully answered how the OS runs processes fast (direct execution, Topic 1) while staying safe (restricted operations, Topic 2) and always in control (timer interrupts, Topic 3) — with the context switch (this topic) as the concrete mechanism tying it all together with the PCB from Module 02.
