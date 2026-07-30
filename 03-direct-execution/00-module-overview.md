# Module 03 — Direct Execution

## Module Goal

By the end of this module, you will understand **the actual low-level mechanism that lets the OS run a process's instructions at full, native hardware speed, while still retaining the ability to safely take the CPU back whenever it needs to** — whether the process cooperates or not. Module 02 described processes and their lifecycle from the outside; this module goes one level deeper, answering: physically, how does the CPU run a process's code directly (for speed), without that process being able to simply refuse to give control back, or do something dangerous while it has the CPU?

## Topics Covered in This Module

1. **[The Limited Direct Execution Protocol](01-the-limited-direct-execution-protocol.md)** — Running a process's instructions directly on the CPU for speed, while the OS still sets the boundaries ahead of time.
2. **[Restricted Operations and Traps](02-restricted-operations-and-traps.md)** — How dangerous instructions are forbidden in user mode, and how the trap table ensures a process can only ever enter the kernel at a safe, predetermined location.
3. **[Timer Interrupts and Regaining Control](03-timer-interrupts-and-regaining-control.md)** — Why the OS cannot simply trust a process to voluntarily give back the CPU, and the hardware timer that guarantees it doesn't have to.
4. **[The Context Switch](04-the-context-switch.md)** — The precise mechanical steps the OS takes to save one process's execution state and load another's, tying together the PCB from Module 02.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 02 in full, especially Topic 3 (The Process Control Block) and Module 01, Topic 6 (System Calls).

## How to Study This Module

Read in order — this module builds one continuous argument. Topic 1 sets up the core tension: run code fast (directly on hardware) vs. keep the OS in control (safety). Topic 2 resolves the "what if a process tries something dangerous" half of that tension using the trap mechanism you first saw conceptually in Module 01, Topic 6, now explained fully. Topic 3 resolves the harder half: "what if a process just never gives the CPU back voluntarily" — the answer (timer interrupts) is one of the most important ideas in this entire course, because it's the concrete reason an OS can never be held hostage by a single misbehaving or infinite-looping program. Topic 4 then shows exactly what happens, instruction by instruction, during the context switch these mechanisms make possible.
