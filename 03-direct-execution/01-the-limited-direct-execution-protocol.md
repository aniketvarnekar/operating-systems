# The Limited Direct Execution Protocol

## Learning Objectives

By the end of this section you should be able to:
- Explain what "direct execution" means and why it's necessary for performance
- Explain what the word "limited" adds to that idea, and why it's necessary for safety
- Describe, at a high level, the full lifecycle of running a process under this protocol

## Prerequisites

- Module 02, Topic 1 (The Process Abstraction)
- Module 01, Topic 6 (System Calls)

## Motivation

Module 01 (Topic 3) promised that CPU virtualization gives every process the illusion of a dedicated CPU. Module 02 described processes and how the OS creates and tracks them. This topic answers the missing mechanical question: when the OS actually lets a process run, does the CPU somehow run an "interpreted," OS-supervised version of that process's instructions — which would be slow — or does it run the process's real, native machine instructions directly? And if it's the latter (for speed), what stops the process from doing something dangerous, or simply never giving the CPU back?

## Problem Statement

The OS wants two things that seem to be in tension:

1. **Performance**: a process's code should run at full native hardware speed — no OS layer should sit "in between" every single instruction, slowing execution down, the way a software interpreter would.
2. **Control**: the OS must still be able to enforce safety (stop a process from touching hardware it shouldn't) and must still be able to take the CPU back from a process, even one that never asks to give it up.

A naive approach to (1) — just let the process's code run directly on the bare CPU, with no OS involvement at all once it starts — completely destroys (2): the process could execute any instruction it wants, including ones that access hardware directly or that simply loop forever without ever returning control to the OS.

## Concept

### Definition

> **Direct execution** means running a process's compiled machine instructions straight on the physical CPU, with no software interpretation layer — the process runs exactly as fast as the hardware allows. **Limited** direct execution adds the crucial qualifier: the OS establishes restrictions and control points *before* handing off the CPU, so that even though the process's normal instructions run unsupervised at full speed, the OS still guarantees it can intervene the moment something dangerous is attempted, or the moment it simply wants the CPU back.

The word "limited" is doing all the interesting work in this term — it's the difference between "the OS trusts the process" (dangerous, and effectively what Module 01, Topic 1 already ruled out) and "the OS lets the process run fast, but only within boundaries it set up in advance and that the hardware itself enforces."

### The High-Level Protocol

At a high level, running a process under limited direct execution looks like this:

1. **Before running anything (at boot time)**: the OS sets up the trap table (Topic 2) and other hardware-enforced boundaries — this only happens once, in kernel mode, before any user process ever runs.
2. **To start the process**: the OS creates its PCB (Module 02, Topic 3), sets up its initial state, and then performs a special transition — often described as "returning from a trap that never actually happened" — that switches the CPU into user mode and jumps to the process's first instruction.
3. **While the process runs**: its ordinary instructions execute directly on the CPU, at full native speed, with zero OS involvement — exactly like direct execution, for exactly as long as nothing exceptional happens.
4. **The moment something needs OS attention** — the process makes a system call (Module 01, Topic 6), attempts a restricted operation, or a hardware timer interrupt fires (Topic 3) — control transfers back into the kernel, which decides what to do next (service the request, terminate the process, switch to a different process, etc.).
5. **Then the loop continues**: the OS eventually returns control to this process (or a different one) via the same "return from trap" mechanism, and direct execution resumes.

## Internal Working (Preview)

```
   OS                                  Hardware                Program (process)
   ──                                  ────────                ─────────────────
   create PCB, set up
   initial register state
   (Module 02, Topic 3)

   "return-from-trap"        ───────►  switch to user mode,
                                        jump to process's
                                        first instruction
                                                                  run instruction 1
                                                                  run instruction 2
                                                                  run instruction 3
                                                                  ... (full native speed,
                                                                       zero OS involvement)

                                        restricted instruction
                                        attempted, OR system
                                        call made, OR timer
                                        interrupt fires
                             ◄───────  trap into kernel
   handle request / decide
   what runs next
   (may resume this process
    or switch to another)
```

## Real-World Analogy

Think of a substitute teacher (the OS) who wants students (processes) to be able to work independently and quickly, without being micromanaged line-by-line — but who has still put clear, unbreakable ground rules in place before stepping out of the room: no leaving the classroom, no touching the supply cabinet, and a loud bell (a timer interrupt) that will ring automatically after exactly ten minutes no matter what, calling the teacher back in regardless of whether any student asked for help. While the bell hasn't rung and no student has raised a hand (a system call), students work completely independently, at their own full pace — direct execution. But the ground rules (restricted operations, Topic 2) and the guaranteed bell (Topic 3) mean the teacher never actually loses control of the room, even without watching every single second.

## Why This Design Is Necessary

Pure direct execution (no restrictions at all) is fast but unsafe — Module 01, Topic 1 already established why unrestricted hardware access can't be trusted to self-police. Pure interpretation (the OS manually simulates every single instruction a process wants to run, checking each one) would be safe but is far too slow for any real-world use — imagine every instruction of every program requiring an OS round-trip. Limited direct execution is the engineering answer that gets (almost) all of direct execution's speed for the overwhelming majority of instructions (ordinary computation, which never needs OS involvement at all), while still guaranteeing safety and control at the specific, well-defined moments that matter (restricted operations and forced interrupts).

## Advantages of This Design

- **Near-native performance** — the vast majority of a process's instructions run with zero OS overhead, since ordinary computation never needs to trap into the kernel at all.
- **Guaranteed control retention** — the OS is never at the mercy of a process's cooperation; both restricted-operation traps (Topic 2) and timer interrupts (Topic 3) are hardware-enforced, not optional.

## Disadvantages / Costs

- **Real overhead exists at trap/interrupt boundaries** — every system call and every timer interrupt still costs real CPU cycles for the mode switch and context save/restore (Topic 4); the goal is minimizing how often and how expensive these are, not eliminating them entirely.
- **Complexity of correct hardware/OS coordination** — this protocol depends on the CPU hardware supporting specific features (privilege modes, a trap table, a programmable timer) that the OS must configure correctly at boot; a mistake here undermines the entire safety model.

## Best Practices

- When evaluating "why is my program's system-call-heavy code slower than expected," remember that each system call incurs a real, non-zero trap/mode-switch cost (Module 01, Topic 6) — this is a direct consequence of the limited direct execution model, not an implementation bug.
- Keep the two halves of "limited" distinct in your head as you move to Topics 2 and 3: restricting *what a process can do on its own* (Topic 2) is a different mechanism from *guaranteeing the OS gets the CPU back* (Topic 3) — they solve two different problems.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The OS runs every instruction of a process itself, one at a time, to keep control." | That would be pure interpretation, which is far too slow for real use. Limited direct execution instead lets the process's own instructions run directly and unsupervised, intervening only at specific, well-defined trap/interrupt points. |
| "Direct execution means the process has unrestricted hardware access, same as if there were no OS at all." | The "limited" part specifically means the OS pre-establishes hardware-enforced boundaries (restricted operations, a forced-return timer) before ever handing off the CPU — it's not unrestricted at all, just fast for the common case. |

## Interview Questions

1. **Q: What does "limited direct execution" mean?**
   A: Running a process's instructions directly on the CPU at full native speed (direct execution), while the OS pre-establishes hardware-enforced restrictions and guaranteed control-transfer points (the "limited" part) so it never loses the ability to intervene or reclaim the CPU.

2. **Q: Why doesn't the OS just interpret/simulate every instruction a process wants to run, to stay fully in control?**
   A: That would make every single instruction pay the cost of OS involvement, which is far too slow for real-world programs. Limited direct execution instead lets the overwhelming majority of instructions run with zero OS overhead, paying a cost only at specific trap/interrupt points.

3. **Q: What are the two main situations under limited direct execution where control transfers back to the OS?**
   A: A process voluntarily invoking a restricted operation via a trap/system call (Topic 2), or a hardware timer interrupt forcibly returning control regardless of what the process is doing (Topic 3).

## Summary

- Limited direct execution runs a process's instructions directly on the CPU for near-native speed, while the OS pre-establishes hardware-enforced boundaries that guarantee it can still intervene.
- The protocol: the OS sets up boundaries once at boot, hands off the CPU via a "return from trap," lets the process run unsupervised, and regains control via a trap or an interrupt.
- This resolves the tension between wanting fast, unsupervised execution and needing to guarantee the OS is never at the mercy of a process's cooperation.
- The next two topics cover each half of "limited" in full mechanical detail: restricted operations and traps (Topic 2), and the timer interrupt that guarantees the OS gets the CPU back even from an uncooperative process (Topic 3).
