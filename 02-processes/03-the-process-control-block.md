# The Process Control Block

## Learning Objectives

By the end of this section you should be able to:
- Explain what a Process Control Block (PCB) is and why the OS needs one per process
- List the categories of information a PCB typically stores
- Explain how the PCB makes switching the CPU between processes possible at all

## Prerequisites

- Topic 1 (The Process Abstraction)
- Topic 2 (Process States and Lifecycle)

## Motivation

Topic 2 described processes moving between Running, Ready, and Blocked — but for the OS to actually pause a Running process and later resume it *exactly* where it left off, it needs somewhere to record everything about that process the instant it stops running. That "somewhere" is the Process Control Block, and it is the single data structure that makes multiprogramming and time-sharing (Module 01, Topic 2) technically possible.

## Problem Statement

Suppose Process A is running, and the OS decides to switch the CPU to Process B (perhaps A's time slice expired, or A blocked on I/O). The instant the CPU switches away from A, all of A's register values — including its program counter, marking exactly which instruction it was about to execute — are still physically sitting inside the one shared set of CPU registers. The moment Process B starts running and uses those same physical registers for its own purposes, A's exact register state would be gone forever unless it was saved somewhere first.

Later, when the OS wants to resume Process A, it needs to restore the CPU's registers to *exactly* the values A had at the moment it was paused — otherwise A would resume from the wrong instruction, or with corrupted data in its working registers, and the illusion of "A has had a dedicated, continuously running CPU the whole time" (Module 01, Topic 3) would be broken.

## Concept

### Definition

> A **Process Control Block (PCB)** — sometimes called a task struct or process descriptor — is a kernel data structure, one per process, that stores everything the OS needs to know about that specific process: enough to pause it at any instant and later resume it with no observable difference, and enough to manage it as a distinct entity throughout its life.

### What a PCB Typically Contains

- **Saved register state**: the exact values of all CPU registers (including the program counter and stack pointer) at the moment this process was last paused — this is precisely what gets restored to resume the process later.
- **Process state**: which of the states from Topic 2 (Running, Ready, Blocked, etc.) this process currently occupies.
- **Process ID (PID)**: a unique identifier the OS and other processes use to refer to this specific process.
- **Memory management information**: pointers to this process's address space structures (page tables, covered starting in Module 08) — everything needed to correctly translate this process's memory addresses.
- **Open file information**: a list/table of files and other resources this process currently has open.
- **Scheduling information**: priority, how much CPU time it's already consumed, and other bookkeeping the scheduler (Modules 04–05) uses to make decisions.
- **Parent/child relationships**: which process created this one, and which processes (if any) this one has created — relevant to Topic 6 (`wait()`).

### The PCB and the Context Switch

The act of the OS saving the currently-running process's register state into its PCB, then loading a *different* process's previously-saved register state from *its* PCB into the CPU, is called a **context switch**. This is the actual mechanism — explained fully in Module 03 — by which the OS makes it possible for many processes to take turns on one physical CPU, each unaware that it was ever paused at all.

## Internal Working (Preview)

```
              CPU is switching from Process A to Process B
              ────────────────────────────────────────────

   Step 1: SAVE                          Step 2: LOAD
   Copy current CPU register        Copy Process B's previously
   values (incl. program counter)   saved register values
   INTO Process A's PCB             FROM Process B's PCB
                                     INTO the CPU registers
   ┌─────────────────┐              ┌─────────────────┐
   │  Process A PCB    │             │  Process B PCB    │
   │  registers: [...] │◄── save     │  registers: [...] │── load ──►  CPU
   │  state: READY      │             │  state: RUNNING    │
   │  PID, open files,   │             │  PID, open files,  │
   │  memory info, ...   │             │  memory info, ...   │
   └─────────────────┘              └─────────────────┘
```

Every process on the system has exactly one PCB. Collectively, the OS's set of all PCBs is effectively its master ledger of "everything currently running or waiting to run on this machine."

## Real-World Analogy

Think of a PCB like an actor's detailed continuity notes on a film set, tracked separately for every actor in a large ensemble production, where scenes for different actors are shot out of order and interleaved throughout the day. Before actor A steps off set (is paused), the script supervisor writes down exactly where A's character left off — which line, what prop A was holding, where A was standing, A's emotional state in that moment. When it's later actor A's turn again, the supervisor's notes let A resume the scene precisely as if no time had passed for their character at all — even though, in reality, hours and many other actors' scenes happened in between. Each actor has their own separate set of continuity notes (their own PCB); mixing up whose notes belong to whom would visibly break the illusion of a continuous, uninterrupted performance.

## Why This Design Is Necessary

Without a PCB (or an equivalent per-process record), the OS would have no way to pause a process and later resume it with byte-for-byte identical register and bookkeeping state — which is the exact guarantee that makes the "you have a private, continuously running CPU" illusion (Module 01, Topic 3) convincing. A single shared register set physically cannot hold more than one process's state at once, so *somewhere* that state must be saved off per-process the instant a process stops running — the PCB is that somewhere.

## Advantages of the PCB Design

- **Enables the illusion of continuous execution** — a process, once resumed, has no way to detect it was ever paused; its registers and state are restored exactly as they were.
- **Centralizes all per-process bookkeeping** — scheduling info, memory info, and open files all live in one well-defined place per process, simplifying the rest of the OS's design.
- **Enables fast, correct context switching** — Module 03 covers the actual switching mechanism, but none of it is possible without a reliable place to save and restore each process's state.

## Disadvantages / Costs

- **Memory overhead** — every process, even one that does very little, requires the kernel to allocate and maintain a PCB, which costs real (though typically small) kernel memory.
- **Context switch cost** — saving and restoring a PCB's worth of register and bookkeeping state on every switch is not free; it's real, measurable overhead (quantified further in Module 03), part of the broader cost of virtualization discussed in Module 01, Topic 3.

## Best Practices

- When reasoning about "how does the OS remember what a paused process was doing," always trace it back to the PCB — it's the concrete answer to nearly every "how does the OS keep track of X per process" question in this course.
- Keep Process ID (PID), not process name or memory address, as your mental "primary key" for identifying a specific process — it's what the OS and its own APIs (Topics 4–6) use to unambiguously refer to one specific process.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The PCB is just for administrative bookkeeping (like the PID) — it doesn't actually matter for running the process." | The PCB stores the process's actual saved register state, including the program counter — without it, the OS could not correctly resume a paused process at all. |
| "Only running processes need a PCB; ready or blocked processes don't." | Every process, in every state, has exactly one PCB for its entire lifetime — a Ready or Blocked process's PCB is precisely what lets the OS know it exists and what to restore once it's scheduled again. |

## Interview Questions

1. **Q: What is a Process Control Block, and why does the OS need one per process?**
   A: A kernel data structure storing everything the OS needs to know about one specific process — its saved register state, its current state (Running/Ready/Blocked), its PID, memory and file information, and scheduling data — needed so the OS can pause and later resume a process with no observable difference.

2. **Q: What specifically gets saved into a PCB during a context switch?**
   A: The currently-running process's full CPU register state, including the program counter (exactly which instruction to resume at) and the stack pointer, along with updating its process state field.

3. **Q: Why can't the OS just leave a paused process's data in the CPU's registers until it's scheduled again?**
   A: Because the CPU has only one physical set of registers, which the next scheduled process needs to use for its own execution — the paused process's exact register values must be copied out to its PCB first, or they would be overwritten and permanently lost.

## Summary

- The PCB is a per-process kernel data structure holding everything needed to pause and later resume that process indistinguishably, plus PID, state, memory, file, and scheduling information.
- A context switch works by saving the currently-running process's register state into its PCB, then loading a different process's saved state from its own PCB into the CPU.
- Without the PCB, the OS could not correctly maintain the illusion (Module 01, Topic 3) that each process has a dedicated, continuously running CPU.
- The next topic moves from "how the OS tracks an existing process" to "how a brand-new process is created in the first place," via the `fork()` system call.
