# Why Threads?

## Learning Objectives

By the end of this section you should be able to:
- Explain two concrete motivations for using threads instead of separate processes
- Explain why creating and switching between threads is typically cheaper than doing the same with processes
- State, at a high level, the key difference between a thread and a process

## Prerequisites

- Module 01, Topic 4 (Concurrency)
- Module 02, Topic 4 (The fork() System Call)

## Motivation

Module 02 gave you one way to get multiple things happening "at once": create multiple separate processes via `fork()`. This topic explains why that's often not the tool you actually want, and introduces the alternative — threads — that Module 01, Topic 4 already previewed conceptually when it first discussed concurrency's core problems.

## Problem Statement

Suppose a program wants to perform several independent, related tasks at once — for example, a web server handling multiple client connections simultaneously, all of which need to share the same in-memory cache of recently-served data. Using separate processes (`fork()`, Module 02, Topic 4) for each connection would work, but each process gets its own **entirely separate address space** (Module 06, Topic 1) — meaning the in-memory cache would need to be duplicated (wasteful) or shared through comparatively heavyweight, explicit inter-process communication mechanisms, rather than simply being one shared piece of data multiple execution contexts can access directly.

## Concept

### The Thread: Multiple Execution Contexts, One Address Space

> A **thread** is an independent sequence of execution — with its own program counter, its own registers, and its own stack — that runs *within* a single process, sharing that process's one address space (code, heap, and global/static data) with any other threads in the same process.

The critical distinction from a process (Module 02, Topic 1): a process gets its own entire, private address space. Multiple threads *within the same process* share that one address space directly — meaning they can read and write the same heap-allocated data, the same global variables, without any of the explicit, heavier-weight communication mechanisms that separate processes would require.

### Motivation 1: Exploiting Multiple CPUs for a Single Program

Recall Module 05, Topic 4's multi-CPU scheduling: modern machines commonly have several CPU cores. A single-threaded program can only ever use *one* core at a time, no matter how many are available. By splitting independent work across multiple threads, a single program can have different threads genuinely running simultaneously on different cores, directly speeding up computation that can be meaningfully divided into independent pieces.

### Motivation 2: Avoiding Blocking on Slow Operations

Recall Module 04, Topic 6's CPU/I/O burst model: a single sequence of execution that issues a slow I/O request blocks entirely until that request completes. A program built from multiple threads can have one thread block on a slow operation (a network request, a disk read) while other threads in the *same* process continue doing useful work — without needing an entirely separate process (and its own address space) just to keep something else running during that wait.

### Why Threads Are Cheaper Than Processes

Creating a new process (via `fork()`, Module 02, Topic 4) requires setting up an entire new address space and PCB (Module 02, Topic 3) — genuinely more setup work than creating a new thread, which reuses the *same* address space as the process it belongs to and only needs its own smaller, per-thread bookkeeping (covered in Topic 2). Similarly, a context switch (Module 03, Topic 4) between two threads of the *same* process can be cheaper than a context switch between two entirely different processes, since address-space-related state (like page table pointers, Module 08, Topic 2) doesn't need to change at all when switching between threads that already share the same address space.

## Internal Working (Preview)

```
   Multiple PROCESSES (Module 02):        Multiple THREADS within ONE process:

   ┌─────────┐  ┌─────────┐          ┌───────────────────────────┐
   │ Process A │  │ Process B │          │        Process              │
   │ own addr   │  │ own addr   │          │  ┌────────┐ ┌────────┐   │
   │ space       │  │ space       │          │  │ Thread 1 │ │ Thread 2 │   │
   └─────────┘  └─────────┘          │  │ own PC,  │ │ own PC,  │   │
     (separate,                       │  │ own stack│ │ own stack│   │
      isolated —                      │  └────────┘ └────────┘   │
      Module 06,                      │                              │
      Topic 1)                        │   SHARED: code, heap,       │
                                        │   global/static data         │
                                        └───────────────────────────┘
```

## Real-World Analogy

Think of a process as an entire independent office building, complete with its own filing cabinets, its own supplies, its own everything — and a thread as one of several employees working *inside* the same office building. Multiple employees (threads) in the same building can walk over to the same shared filing cabinet (the heap/global data) and pull the same file directly — fast, and no need to photocopy anything across buildings. Two employees in two entirely *different* office buildings (two separate processes) can't just walk over to each other's filing cabinets at all; if they need to share information, they'd have to formally mail copies back and forth (heavier-weight inter-process communication) — much slower, and each building needs its own full set of supplies and infrastructure in the first place (the cost of creating a whole new process vs. just hiring one more employee into an existing building).

## Why This Design Is Necessary

Some problems are naturally structured as several independent, related pieces of work that all need access to the same shared data — a shared cache, a shared connection pool, shared program state. Forcing this into separate processes (with entirely separate address spaces) either duplicates that shared data wastefully or requires explicit, comparatively expensive communication mechanisms just to keep it in sync. Threads solve this directly by making the sharing the *default*, since all threads within a process already sit inside the exact same address space — no special mechanism is needed at all for shared data to be visible to every thread.

## Advantages of Threads

- **Cheaper creation and context-switching** than separate processes, since address-space setup and switching overhead (Module 08, Topic 2's page tables) is largely avoided between threads of the same process.
- **Direct, natural data sharing** — no explicit inter-process communication mechanism is needed for threads to read and write the same heap or global data.
- **Exploits multiple CPU cores** for a single logical program, and lets useful work continue on other threads while one thread blocks on slow I/O.

## Disadvantages / New Risks (Previewed)

- **Shared data means shared danger** — because threads directly share the same address space, uncoordinated access to shared data between threads is precisely the race-condition risk Module 01, Topic 4 introduced, and precisely what Modules 13–15 build tools to manage.
- **A bug in one thread can more easily corrupt the whole process** — since there's no address-space isolation between threads of the same process (unlike between separate processes, Module 06, Topic 1), a wild write from one thread can directly corrupt data another thread in the same process depends on.

## Best Practices

- Reach for threads specifically when the work genuinely needs to share data directly and frequently, or when exploiting multiple cores for one logical program is the goal; reach for separate processes when strong isolation between the pieces of work is more important than cheap sharing.
- Keep firmly in mind, going into Module 13, that "threads share an address space" is simultaneously threads' greatest strength (Motivation 1–2 above) and the direct root cause of every concurrency bug covered later in this course.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A thread and a process are basically the same thing, just with a different name." | A process has its own entire, private address space (Module 06, Topic 1); a thread runs within an existing process's address space, sharing it directly with any other threads in that same process — a fundamentally different sharing model, not just a naming difference. |
| "Threads are strictly better than processes in every situation." | Threads' direct data sharing is exactly what makes uncoordinated access between them dangerous (Module 01, Topic 4's race conditions), and a single thread's bug can corrupt the entire process it belongs to, since there's no address-space isolation between threads of the same process. |

## Interview Questions

1. **Q: What is a thread, and how does it differ from a process?**
   A: A thread is an independent sequence of execution — with its own program counter, registers, and stack — running within a process, sharing that process's single address space with any other threads in it. A process, by contrast, has its own entirely separate, private address space.

2. **Q: Give two concrete motivations for using threads instead of separate processes.**
   A: Exploiting multiple CPU cores for a single logical program (different threads can run simultaneously on different cores), and avoiding having the entire program block when one part is waiting on slow I/O (other threads can continue useful work while one thread is blocked).

3. **Q: Why is creating a thread generally cheaper than creating a new process?**
   A: A new process requires setting up an entirely new address space and PCB; a new thread reuses the existing process's address space and only needs smaller, per-thread bookkeeping, avoiding the more expensive address-space setup work.

## Summary

- A thread is an independent sequence of execution within a process, sharing that process's single address space with any other threads in it — unlike a process, which has its own entirely private address space.
- Threads are motivated by exploiting multiple CPU cores for one program, and by letting useful work continue on other threads while one thread blocks on slow I/O.
- Threads are cheaper to create and switch between than processes, since address-space setup and switching overhead is largely avoided.
- The shared address space that makes threads powerful is also the direct root cause of the race-condition risks Module 01, Topic 4 introduced — the next topic covers exactly what's shared and what's private per thread, setting up Modules 13–15's concurrency tools.
