# Module 03 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Limited Direct Execution Protocol** — running instructions directly for speed, while the OS pre-establishes control boundaries
- [x] **Restricted Operations and Traps** — the boot-time trap table, and why a process can request a service but never choose where the hardware jumps
- [x] **Timer Interrupts and Regaining Control** — why cooperative CPU sharing fails, and how a hardware timer guarantees the OS always regains control
- [x] **The Context Switch** — the precise save/decide/load/return sequence, the kernel stack, and the mode-switch vs. context-switch distinction

## The Big Picture

This module answered the mechanical question Module 02 left open: how does the OS run many processes on one physical CPU, quickly and safely, without ever being at the mercy of any single process's behavior?

```
       LIMITED DIRECT EXECUTION, END TO END
       ─────────────────────────────────────

  BOOT TIME:  OS registers trap table + starts hardware timer
              (both fully configured before any user process runs)

  RUN TIME:   process runs directly on CPU, full native speed
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
  voluntary trap          timer interrupt fires
  (syscall, Topic 2)      (forced, Topic 3)
        │                       │
        └───────────┬───────────┘
                     ▼
           control returns to kernel
        (mode switch — always happens)
                     │
                     ▼
       scheduler decides: same process,
       or a different one? (Modules 04–05)
                     │
              ┌──────┴──────┐
              ▼             ▼
        resume same    context switch to
        process        a different process
        (Topic 4, no   (Topic 4, full PCB
         PCB swap)      save/load)
```

## Practical Connections

- **Why a runaway or infinite-looping program never permanently freezes your whole computer** — the timer interrupt (Topic 3) guarantees the OS always regains the CPU, letting you still move your mouse, switch windows, or force-quit the offending program.
- **Why heavy system-call usage (e.g., excessive small file reads) shows up as real, measurable overhead in profiling tools** — every trap costs a mode switch (Topic 2), and this module explained exactly why that cost is real and unavoidable, not a measurement artifact.
- **Why "context switch overhead" is a real line item operating systems and performance engineers actively try to minimize** — Topic 4's step-by-step trace shows concretely what work a context switch actually performs, and why doing it too often wastes CPU time on switching rather than useful work.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Direct execution vs. limited direct execution | Direct execution alone is fast but unsafe (no restrictions at all). "Limited" adds pre-established, hardware-enforced boundaries (the trap table, the timer) so the OS never loses ultimate control. |
| Cooperative vs. non-cooperative (preemptive) CPU sharing | Cooperative relies on processes voluntarily yielding/trapping — and fails against an uncooperative process. Preemptive uses a hardware timer interrupt that fires regardless of process behavior, guaranteeing control is always regained. |
| Mode switch vs. context switch | A mode switch (user↔kernel) happens on every single trap or interrupt. A context switch — the more expensive full save/load of a different process's state — only happens when the scheduler decides to run someone else. |

## What's Next

Module 03 established that the OS can always, unconditionally, regain the CPU — but deliberately left one question unanswered: once the OS has the CPU back, and has a choice among several ready processes, **which one should it actually choose to run next, and for how long?** That is a policy question, not a mechanism question, and it's the subject of **Module 04 — CPU Scheduling Basics**, starting with the metrics used to judge a scheduling policy and the classic algorithms (FIFO, SJF, STCF, Round Robin) built around them.
