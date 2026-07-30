# The Thread Abstraction

## Learning Objectives

By the end of this section you should be able to:
- List precisely what is shared between threads of the same process, and what is private to each
- Explain why each thread needs its own stack, even though the heap is shared
- Explain what a Thread Control Block (TCB) is, and how it relates to the PCB from Module 02

## Prerequisites

- Topic 1 (Why Threads?)
- Module 02, Topic 3 (The Process Control Block)
- Module 06, Topic 1 (The Address Space Abstraction)

## Motivation

Topic 1 established that threads share an address space — but "the address space" is made up of several distinct regions (Module 06, Topic 1), and not all of them can safely be shared in exactly the same way. This topic pins down precisely what's shared and what must remain private per thread, and why — the single most important distinction for understanding every concurrency bug covered starting in Module 15.

## Problem Statement

If two threads share the same heap and the same global variables, can they also safely share a single stack? Recall from Module 02, Topic 1 that a stack holds function call frames, local variables, and return addresses — and that a running sequence of execution needs to track exactly where it currently is (its program counter) and what its local, in-progress computation state looks like (its registers). If two threads tried to use the *same* stack and the *same* registers simultaneously, could either of them make any independent progress at all?

## Concept

### What Is Shared Between Threads of the Same Process

Because all threads within one process run inside that process's single address space (Topic 1), they share:

- **Code (Text)**: the same compiled instructions — any thread can call any function in the program.
- **Heap**: the same dynamically-allocated memory (Module 06, Topic 2) — one thread's `malloc()`'d data is directly visible to every other thread in the same process.
- **Static/Global Data**: the same global variables — a change made by one thread is immediately visible to all others.
- **Open file descriptors and other process-wide OS resources**: the same set, inherited from the shared process.

### What Remains Private to Each Thread

Despite sharing an address space, each individual thread needs its own:

- **Program counter**: each thread is independently executing its own sequence of instructions — they cannot share one single "current position," since each is genuinely at a different point in the program at any given moment.
- **Registers**: each thread's in-progress computation (values being actively worked on, right now) must be distinct from every other thread's — exactly analogous to why each process needs its own saved register state (Module 02, Topic 3), just at a finer granularity, per thread instead of per process.
- **Stack**: each thread needs its own independent stack for its own function calls, local variables, and return addresses — sharing one stack across threads would mean one thread's function call could overwrite another's local variables or return address, making independent execution impossible.

This means, structurally, that each thread's own stack lives *within* the process's shared address space, but is treated as that one specific thread's own private region — carved out of the overall address space, but not shared with other threads the way the heap and global data are.

### The Thread Control Block (TCB)

Recall the Process Control Block (Module 02, Topic 3): a per-process kernel structure holding everything needed to pause and resume that process. Threads need an analogous, finer-grained structure:

> A **Thread Control Block (TCB)** holds everything the OS needs to save and restore one specific thread's execution state — its saved registers (including the program counter) and a pointer to its own private stack — separate from the PCB-level information (like open files and memory-management structures) that's shared across every thread belonging to the same process.

A process with multiple threads, then, has **one PCB** (shared, process-wide information: address space, open files) and **multiple TCBs** (one per thread, each with its own saved registers and stack pointer) — the exact same context-switching machinery from Module 03, Topic 4 applies here too, just now switching between threads (potentially within the same process) rather than only between entirely separate processes.

## Internal Working (Preview)

```
   ONE Process's Address Space (shared by all its threads):

   ┌─────────────────────────────────────────────┐
   │  Code (shared)                                  │
   │  Static Data (shared)                            │
   │  Heap (shared) ↓                                  │
   │                                                     │
   │  Thread 1's private stack ↑    (grows down from  │
   │  Thread 2's private stack ↑     separate reserved │
   │  Thread 3's private stack ↑     regions)          │
   └─────────────────────────────────────────────┘

   Per-thread (NOT shared):          Per-process (shared, one PCB):
   ┌────────────┐                  ┌───────────────────┐
   │ TCB: Thread 1 │                  │  PCB: address space,   │
   │  registers,    │                  │  open files, etc.       │
   │  own stack ptr │                  └───────────────────┘
   ├────────────┤
   │ TCB: Thread 2 │
   │  registers,    │
   │  own stack ptr │
   └────────────┘
```

## Real-World Analogy

Recall Topic 1's office-building analogy: if the building is the process, and employees are threads, then the shared filing cabinets (heap, global data) and the shared building infrastructure (code, open resources) are things every employee can access directly. But each employee still needs their own personal desk (a private stack) to keep their own current, in-progress paperwork organized — if two employees tried to share one single desk simultaneously, one's in-progress documents would constantly get shuffled, misplaced, or overwritten by the other's. And each employee has to be, at any given moment, personally standing at their own specific point in their own specific task (their own program counter) — "where employee 1 currently is in their work" and "where employee 2 currently is" are simply different facts that can't be merged into one.

## Why This Design Is Necessary

Sharing the heap and global data is precisely what makes threads useful (Topic 1) — but a thread's stack and registers represent its own, individually-in-progress computation, which by definition must be distinguishable from every other thread's in-progress computation for either one to make independent sense at all. Without a private stack and private registers per thread, there would be no way to represent "thread 1 is in the middle of function X, about to do Y" as a fact distinct from "thread 2 is in the middle of function Z, about to do W" — independent execution simply requires this much private state, no matter how much else is shared.

## Advantages of This Design

- **Enables genuinely independent execution** while still allowing direct, efficient sharing of the data that actually benefits from being shared (heap, globals).
- **Reuses established context-switching machinery** (Module 03, Topic 4) at a finer granularity, rather than requiring an entirely new mechanism for switching between threads.

## Disadvantages / Risks (Previewed)

- **The shared parts are exactly where danger lives** — because the heap and global data are directly, simultaneously accessible to every thread with no automatic coordination, uncoordinated concurrent access to them is precisely the race-condition risk from Module 01, Topic 4, requiring the explicit tools covered in Modules 13–14.
- **A bug in one thread's stack/register usage (e.g., a buffer overflow, Module 06, Topic 3) can still, in principle, corrupt adjacent memory used by another thread**, since there's no hardware-enforced isolation between threads within the same address space, unlike the isolation between separate processes (Module 06, Topic 1).

## Best Practices

- When debugging multithreaded code, always ask "is this specific piece of data shared (heap/global) or private (stack-local)?" as the very first diagnostic question — shared data is where race conditions can occur; purely thread-local stack data cannot be directly corrupted by another thread's ordinary execution.
- Keep the PCB-vs-TCB distinction precise: one PCB per process (shared, process-wide resources), one TCB per thread (private execution state) — this maps directly onto "what's shared" vs. "what's private" from this topic.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "All of a process's memory, including each thread's stack, is shared equally among all its threads." | While all threads share the same overall address space, each thread's own stack is treated as that thread's private region within it — sharing a stack across threads would make independent function calls and local variables impossible to maintain correctly. |
| "Threads need their own PCB, just like processes do." | Threads share one PCB (process-wide information) with all other threads in the same process; each thread instead has its own, smaller TCB (register state and stack pointer) — a finer-grained structure than a full PCB. |

## Interview Questions

1. **Q: What do threads of the same process share, and what remains private to each thread?**
   A: They share the process's code, heap, and global/static data, along with process-wide resources like open files. Each thread keeps its own private program counter, registers, and stack, since each represents an independently-progressing sequence of execution.

2. **Q: Why does each thread need its own private stack, even though the heap is shared?**
   A: A stack holds function call frames, local variables, and return addresses for one specific, independently-executing sequence — sharing one stack across threads would let one thread's function calls overwrite another's local variables or return address, making independent execution impossible.

3. **Q: What is a Thread Control Block (TCB), and how does it relate to a Process Control Block (PCB)?**
   A: A TCB holds one thread's saved execution state — its registers (including the program counter) and its stack pointer. A process has one PCB (shared, process-wide information like the address space and open files) but potentially multiple TCBs, one per thread, each capturing that specific thread's own private execution state.

## Summary

- Threads of the same process share code, heap, and global/static data — direct, efficient sharing with no special mechanism needed.
- Each thread keeps its own private program counter, registers, and stack, since these represent its own independently-progressing execution.
- A Thread Control Block (TCB) holds one thread's private execution state, while one PCB continues to hold process-wide, shared information for all its threads.
- The shared heap and global data are exactly where uncoordinated concurrent access becomes dangerous — the next topic covers the concrete API for creating and managing threads, and Modules 13–15 build the tools needed to make shared-data access safe.
