# Process States and Lifecycle

## Learning Objectives

By the end of this section you should be able to:
- Name the three core process states and what each one means
- Explain exactly which events trigger a transition between each pair of states
- Explain why a process can be "ready to run" yet not actually be running

## Prerequisites

- Topic 1 (The Process Abstraction)

## Motivation

A single physical CPU (or a small, fixed number of cores) must serve many processes, most of which are not actually executing at any given instant. To reason correctly about scheduling (Modules 04–05), you first need a precise vocabulary for "what is this process doing right now, if not literally running on the CPU" — that vocabulary is process state.

## Problem Statement

At any given moment on a typical computer, dozens or hundreds of processes exist, but there are only a handful of physical CPU cores. Clearly, only a handful of processes can be *actually executing instructions* at any one instant. So what are all the other processes doing? Are they all identical in status ("not running"), or are there meaningfully different situations a non-running process can be in — for example, a process that's ready to go the instant it gets a turn, versus a process that couldn't run right now even if you gave it the CPU?

## Concept

### The Three Core States

> A process, at any moment, is in exactly one of three core states:
>
> - **Running**: the process is currently executing instructions on a CPU, right now, this instant.
> - **Ready**: the process is prepared to run — it has everything it needs — but the OS has not currently scheduled it onto a CPU. It's waiting purely for its turn.
> - **Blocked**: the process cannot usefully run right now, even if given a CPU, because it's waiting on some event to occur — most commonly, waiting for a slow I/O operation (a disk read, a network response) to complete.

The critical distinction to internalize is between **ready** and **blocked**: both are "not currently running," but for entirely different reasons. A ready process is being deliberately held back purely by the scheduler's choice — give it the CPU right now, and it would immediately make useful progress. A blocked process, even if handed the CPU this instant, has nothing useful to do yet — it's waiting on something entirely outside the CPU's control (like a disk finishing a read).

### State Transitions

```
                    scheduled (dispatch)
        ┌───────────────────────────────────────┐
        │                                        ▼
   ┌─────────┐                              ┌─────────┐
   │  READY   │◄────────────────────────────│ RUNNING  │
   └─────────┘   descheduled (preempted /    └─────────┘
        ▲         time slice expired)              │
        │                                            │ I/O request issued
        │ I/O completes                              ▼
        │                                       ┌─────────┐
        └───────────────────────────────────────│ BLOCKED  │
                                                  └─────────┘
```

- **Ready → Running**: the scheduler (Modules 04–05) picks this process and gives it a CPU. This transition is called being *dispatched* or *scheduled*.
- **Running → Ready**: the OS takes the CPU away from this process even though it could keep running — usually because its time slice expired, or a higher-priority process needs to run. This is called being *preempted*.
- **Running → Blocked**: the process itself issues a request (e.g., reading a file) that cannot complete immediately, so it voluntarily gives up the CPU until that request is satisfied.
- **Blocked → Ready**: the event the process was waiting on finally occurs (e.g., the disk read completes) — the process becomes eligible to run again, but doesn't necessarily get the CPU immediately; it re-enters the ready pool to await its next turn.

Notice there is **no direct arrow from Blocked to Running** — a process that was waiting on an event must always pass back through Ready first; it cannot skip straight from "waiting on I/O" to "actively executing on the CPU" without first being scheduled.

### Two Additional States

Real operating systems typically add two more states around the edges of a process's life:

- **New/Created**: the process is being set up (its address space and process control block are being initialized) but hasn't yet been admitted to the ready pool.
- **Terminated/Zombie**: the process has finished executing, but the OS retains some of its bookkeeping (notably its exit status) until its parent explicitly collects it — covered in full in Topic 6 (`wait()`).

## Real-World Analogy

Think of customers at a single-checkout-lane grocery store:

- **Running**: the customer currently being rung up at the register.
- **Ready**: customers standing in line, fully prepared to check out the instant it's their turn — nothing is stopping them except waiting for their turn.
- **Blocked**: a customer who stepped out of line to go find one more item they forgot — even if the cashier called them up right now, they genuinely couldn't check out yet; they're waiting on something other than just "their turn."

Crucially, a customer who finishes finding their forgotten item (I/O completes) doesn't teleport straight to the register — they rejoin the back of the line (Blocked → Ready), waiting for their turn like everyone else.

## Why This Model Is Designed This Way

Distinguishing Ready from Blocked is what makes efficient CPU virtualization (Module 01, Topic 3) possible at all: if the OS only tracked "running vs. not running," it would have no way to know that a blocked process is a poor candidate to schedule next (it has nothing useful to do yet), while a ready process is an excellent one. This distinction is exactly what allowed multiprogramming (Module 01, Topic 2) to keep the CPU busy — by scheduling a *ready* process the instant the *running* one becomes *blocked* on I/O, instead of leaving the CPU idle.

## Advantages of This State Model

- **Efficient CPU use** — the scheduler can immediately identify which processes are actually worth running (Ready) versus which would be wasted CPU time (Blocked).
- **A clean, universal vocabulary** — "what is this process doing" always reduces to one of a small, well-defined set of states, regardless of what kind of program it is.

## Disadvantages / Limitations

- **The three-state model is a simplification** — real OS implementations often have several additional, more granular sub-states (e.g., different flavors of blocked/sleeping) for internal bookkeeping and debugging purposes, layered on top of this same core idea.

## Best Practices

- When debugging "why is my program slow," first ask which state it's spending most of its time in: if it's mostly Ready, the machine is CPU-contended (too many other processes competing); if it's mostly Blocked, it's most likely waiting on slow I/O, not CPU availability at all — these point to entirely different fixes.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Ready and Blocked are the same thing — both just mean 'not running'." | Ready means the process could usefully run right now if given a CPU; Blocked means it has nothing useful to do yet regardless of CPU availability, because it's waiting on an external event like I/O completion. |
| "A process goes directly from Blocked back to Running the instant its I/O finishes." | It transitions to Ready first; getting an actual CPU turn again still depends on the scheduler, exactly like any other ready process. |

## Interview Questions

1. **Q: What are the three core process states, and what does each mean?**
   A: Running (actively executing on a CPU right now), Ready (able to run immediately, just waiting for the scheduler to grant a CPU), and Blocked (unable to usefully run yet regardless of CPU availability, because it's waiting on an external event like I/O).

2. **Q: What's the practical difference between a Ready process and a Blocked process?**
   A: A Ready process would make immediate, useful progress if handed a CPU right now — it's being held back purely by scheduling choice. A Blocked process has nothing useful to do even with a CPU, because it's waiting on something outside CPU control, like a disk operation finishing.

3. **Q: Can a process transition directly from Blocked to Running, skipping Ready?**
   A: No — when the event it was waiting on completes, it moves to Ready first, and only becomes Running once the scheduler subsequently chooses to dispatch it, just like any other ready process.

## Summary

- A process is always in exactly one of three core states: Running, Ready, or Blocked (plus edge states like New and Terminated).
- Ready means "could run right now, just waiting for a turn"; Blocked means "can't usefully run yet, waiting on an external event."
- Transitions are triggered by specific events: scheduling (Ready→Running), preemption (Running→Ready), an I/O request (Running→Blocked), and I/O completion (Blocked→Ready) — never directly Blocked→Running.
- This state model is exactly what lets an OS keep the CPU busy across many processes, which is the whole point of multiprogramming (Module 01, Topic 2) and CPU scheduling (Modules 04–05).
