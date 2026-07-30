# The Address Space Abstraction

## Learning Objectives

By the end of this section you should be able to:
- Define an address space and list its typical regions
- Explain the three explicit goals memory virtualization must satisfy: transparency, efficiency, and protection
- Explain why every process needs the illusion of starting at address zero, even though physical memory is shared

## Prerequisites

- Module 01, Topic 3 (Virtualization)
- Module 02, Topic 1 (The Process Abstraction)

## Motivation

Modules 02–05 covered CPU virtualization in depth: the process abstraction, the mechanism that runs processes safely, and the policies that decide who runs when. This topic starts the exact same journey for memory — and just like "process" was the key abstraction for the CPU, "address space" is the key abstraction for memory. Get this one right, and Modules 07–11 (translation mechanisms, TLBs, swapping, page replacement) will all read as engineering answers to a goal you already understand.

## Problem Statement

A computer has one, physically shared pool of RAM. Multiple processes run "at once" (Module 02), and each one needs somewhere to store its instructions, its data, and its growing/shrinking stack of function calls. If every process directly used real physical memory addresses, several serious problems appear immediately:

- Two processes could easily be compiled or written assuming they'll use overlapping physical addresses, corrupting each other's data the instant both run at once.
- A buggy process could read or write another process's memory directly, with nothing stopping it — a complete breakdown of the isolation Module 01, Topic 1 already established as essential.
- Every program would need to be written with exact knowledge of which specific physical addresses happen to be free at the moment it runs — utterly impractical, since that depends on which other programs happen to be running at the same time.

## Concept

### Definition

> An **address space** is the OS's abstraction of memory for a running process — a private, contiguous-feeling range of addresses, starting at zero, that a process uses for all of its memory references, entirely independent of where its data actually, physically resides in RAM.

Every process gets its own address space, and (by default) has no visibility into any other process's address space at all — a direct, memory-specific instance of the isolation goal from Module 01, Topic 1.

### The Regions of a Typical Address Space

A process's address space is conventionally organized into a small number of regions, each serving a distinct purpose:

```
   0  ┌─────────────────────┐
      │   Code (Text)         │  ← the process's compiled instructions (typically fixed size)
      ├─────────────────────┤
      │   Static Data          │  ← global/static variables (fixed size)
      ├─────────────────────┤
      │                         │
      │   Heap                  │  ← dynamically allocated memory (grows downward, ↓)
      │       ↓                 │     (Topic 2 covers the API for this)
      │                         │
      │      (free space)       │
      │                         │
      │       ↑                 │
      │   Stack                 │  ← function call frames, local variables (grows upward, ↑)
      │                         │     (Module 02, Topic 1 introduced this)
 MAX  └─────────────────────┘
```

The Heap and Stack deliberately grow **toward each other**, from opposite ends of the free space between them — this lets either one grow as large as it needs to, up to whatever the other one isn't currently using, rather than pre-committing a fixed maximum size to each ahead of time.

### The Three Goals of Memory Virtualization

Just as CPU virtualization (Modules 02–05) had explicit goals (turnaround time, response time, fairness — Module 04, Topic 1), memory virtualization is explicitly judged against three goals:

1. **Transparency**: a process should be entirely unaware that its memory is being virtualized at all — it should behave as if it has sole, private access to its own memory, starting at address zero, exactly as the diagram above shows, regardless of where its data actually lives in physical RAM or how many other processes are also running.
2. **Efficiency**: the virtualization must be achieved without unacceptable costs in time (every single memory access needs some form of address translation — Module 07 onward — and this must be fast) or space (the bookkeeping structures used for translation shouldn't themselves consume an excessive amount of memory).
3. **Protection**: no process should be able to access or affect the memory contents of another process (or the OS itself) without explicit permission — this is the memory-specific instance of the isolation guarantee Module 01, Topic 1 established as a core OS responsibility.

These three goals directly parallel the challenges CPU virtualization faced: transparency mirrors the illusion of a dedicated CPU (Module 01, Topic 3); efficiency mirrors the real overhead of context switches (Module 03, Topic 4) and scheduling decisions; protection mirrors why user/kernel mode and restricted operations exist at all (Module 03, Topic 2).

## Internal Working (Preview)

```
   Process A's view              Process B's view             Physical RAM (shared,
   (its own address space)       (its own address space)       actual location varies)
   ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────────┐
   │ 0: Code            │          │ 0: Code            │          │  ... A's code here ... │
   │    Static Data      │          │    Static Data      │          │  ... B's code here ... │
   │    Heap ↓            │          │    Heap ↓            │          │  ... A's heap here ...  │
   │    Stack ↑           │          │    Stack ↑           │          │  ... B's stack here ...  │
   └─────────────────┘          └─────────────────┘          └─────────────────────┘
   Both processes believe they start at address 0 — the OS (with hardware
   assistance — Modules 07–09) transparently maps each process's own
   addresses to wherever its data actually resides in shared physical RAM.
```

## Real-World Analogy

Think back to the office-building analogy from Module 01, Topic 1: the address space is like every tenant's office being labeled, from their own perspective, as simply "my office," with its own front door, its own desk, its own filing cabinet — regardless of which physical floor or wing of the building they're actually assigned to. Two tenants can both refer to "my desk" without any confusion or conflict, because each tenant's own numbering scheme is private to them; the building management (the OS, with the hardware's help) is the only one that knows the true, physical floor and room number each tenant's "my desk" actually corresponds to, and it's careful never to assign the same physical space to two tenants at once.

## Why This Design Is Necessary

Without the address space abstraction, programs would need to be written with exact, advance knowledge of which physical memory addresses are free at the moment they happen to run — information that depends entirely on what other processes are running simultaneously, and that changes from one run to the next. The address space abstraction removes this dependency completely: every process is written and compiled as if it alone owns all of memory, starting at zero, and the OS (with hardware assistance, Modules 07–09) handles the actual, ever-changing mapping to real physical memory transparently, satisfying isolation without burdening every individual program with the responsibility of avoiding collisions itself.

## Advantages of the Address Space Abstraction

- **Simplicity for programmers** — every program can be written and compiled assuming it owns all of memory from address zero, with zero awareness of other processes.
- **Isolation** — one process cannot access another's memory (or the OS's own) without explicit, deliberate permission, satisfying the protection goal directly.
- **Flexibility in physical placement** — the OS can place a process's actual data anywhere in physical RAM (or even move it later, or temporarily out to disk — Module 10) without that process's own code ever needing to change.

## Disadvantages / Costs

- **Translation overhead** — every single memory reference a process makes must be translated from its private virtual address to a real physical address, which costs real time (mitigated by hardware assistance, particularly the TLB — Module 09) and requires the OS to maintain real bookkeeping structures (page tables — Module 08) that themselves consume memory.

## Best Practices

- When reasoning about "where does my program's data actually live," always distinguish the virtual address your own code sees (Topic 1 of this module) from the physical address it's translated to (Modules 07–09) — conflating the two is one of the most common sources of confusion in this entire course.
- Keep the three goals (transparency, efficiency, protection) as your evaluation checklist for every mechanism introduced from here through Module 11 — each one exists specifically to satisfy one or more of these goals.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Every process's memory addresses correspond directly to the same physical RAM locations." | Each process has its own private, virtual address space, starting at zero — the OS (with hardware help) transparently translates these virtual addresses to wherever the data actually resides in physical RAM, which can differ for every process and can even change over time. |
| "The address space abstraction is only about giving each process 'its own' memory — efficiency isn't really a design concern." | Efficiency is an explicit, first-class goal alongside transparency and protection — the whole abstraction would be useless in practice if the translation machinery required to maintain it were too slow or too memory-hungry to be worth the benefit. |

## Interview Questions

1. **Q: What is an address space?**
   A: The OS's abstraction of memory for a process — a private, contiguous-feeling range of addresses starting at zero that the process uses for all its memory references, entirely independent of where that data physically resides in RAM.

2. **Q: What are the three explicit goals memory virtualization must satisfy?**
   A: Transparency (a process shouldn't be aware its memory is virtualized at all), efficiency (translation must be fast, and its bookkeeping structures shouldn't consume excessive memory), and protection (no process can access another's memory, or the OS's, without explicit permission).

3. **Q: Why do the heap and stack grow toward each other from opposite ends of a process's address space?**
   A: So that either region can grow as large as needed, up to whatever the other isn't currently using, rather than the OS having to pre-commit a fixed maximum size to each ahead of time.

## Summary

- An address space is memory's equivalent of the process abstraction — a private, zero-based range of addresses each process uses, regardless of where its data physically resides.
- It's conventionally organized into Code, Static Data, Heap (grows downward), and Stack (grows upward), with the Heap and Stack growing toward each other through shared free space.
- Memory virtualization is explicitly judged against three goals: transparency, efficiency, and protection — directly paralleling the challenges CPU virtualization faced in Modules 02–05.
- The next topic gets concrete: the actual malloc/free API a C program uses to request and release memory within its own heap region.
