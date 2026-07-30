# Virtualization

## Learning Objectives

By the end of this section you should be able to:
- Define virtualization as the OS applies it to the CPU and to memory
- Explain the illusion each virtualized resource creates, and why that illusion is useful
- Identify which later modules are direct deep dives into CPU virtualization vs. memory virtualization

## Prerequisites

- Topic 1 (What Is an Operating System?)

## Motivation

"Virtualization" is the single word that explains Modules 02 through 11 of this course — nearly half of everything you're about to learn. If this idea clicks now, scheduling policies, address spaces, and page tables will all feel like specific engineering answers to one already-familiar question, instead of a dozen unrelated new topics.

## Problem Statement

A computer physically has exactly **one CPU** (or a small, fixed number of CPU cores) and a **fixed amount of physical RAM**. Yet a modern laptop with a 4-core CPU and 16 GB of RAM routinely runs *hundreds* of programs "at the same time" — a browser with dozens of tabs, a code editor, a music player, background services — each one behaving as if it has the entire machine to itself.

How is that possible? There are nowhere near enough physical CPU cores or physical memory bytes for every single one of those programs to literally, simultaneously have its own private core and its own private bank of RAM.

## Concept

### Definition

> **Virtualization** is the general technique of taking a single, scarce physical resource (like one CPU, or one bank of physical memory) and transforming it — through OS software, often with hardware assistance — into the illusion of a virtual version of that resource for every program that needs one, such that each program can act as if it has that resource entirely to itself.

The word "virtual" is doing real work here: it means "not physically true, but functionally indistinguishable from true, for anyone relying on it."

### Virtualizing the CPU

The OS gives every running program the illusion that it has its own private CPU, executing its instructions continuously. In reality, the OS rapidly switches one (or a few) physical CPU(s) between many programs, running each for a short slice of time before switching to the next.

- The *mechanism* that makes this switching physically possible is called **limited direct execution** (Module 03).
- The *policy* that decides which program gets to run next, and for how long, is called **CPU scheduling** (Modules 04–05).

### Virtualizing Memory

The OS gives every running program the illusion that it has its own private, dedicated block of memory — starting at address zero, as large as it needs — called an **address space**. In reality, physical RAM is one shared pool, and many programs' data is scattered across it (and sometimes even temporarily moved out to disk).

- The abstraction itself, and why a program needs this illusion, is covered in Module 06.
- The mechanism that translates a program's *virtual* addresses into real *physical* RAM locations, transparently and on every single memory access, is called **address translation** — first via simple schemes (Module 07), then via the dominant modern technique, **paging** (Modules 08–09).
- What happens when the illusion of "abundant memory" needs to be stretched *beyond* what physical RAM can actually hold is covered in Modules 10–11.

## Internal Working (Preview)

```
                    ┌───────────────────────────────┐
    Program A       │                               │
   (believes it     │                               │
    owns the        │      Operating System         │
    whole CPU        │   (virtualizes CPU & memory)  │
    and all memory)  │                               │
                     │                               │
    Program B        │                               │
   (believes the     └───────────────┬───────────────┘
    exact same                        │
    thing)                            ▼
                          ┌─────────────────────────┐
                          │   Physical Hardware      │
                          │  (1 CPU core, N bytes    │
                          │   of physical RAM)       │
                          └─────────────────────────┘
```

Both Program A and Program B are told (in effect) "you have the whole CPU and all the memory you asked for" — and neither can tell, from the inside, that this is a carefully maintained illusion.

## Real-World Analogy

Think of a time-share vacation property. Many different families "own" the same physical villa — each family gets a scheduled window of exclusive-feeling use, and during their window, they experience it exactly as if they were the sole owner: the furniture is where they left it, nothing is missing, nothing looks touched by anyone else. The property management company is the "virtualizer" — it handles turnover, cleaning, and scheduling so convincingly that from inside any single family's window, the illusion of sole ownership is essentially perfect, even though a dozen other families use the exact same physical villa throughout the year.

## Why Virtualization Is Designed This Way

The alternative — giving every program direct, exclusive, permanent access to the one physical CPU or a fixed chunk of physical RAM — simply doesn't scale past one program at a time (this was, literally, how early batch-processing computers worked; see Topic 2). Virtualization is the only way to run more programs than you have physical CPUs or physical memory chips, while still letting each program be written as if it doesn't have to think about any other program's existence at all — a massive simplification for every application programmer.

## Advantages of Virtualization

- **Programs are simple to write** — an application never has to coordinate with other applications over CPU time or memory addresses; the OS handles that entirely.
- **Isolation** — because each program only sees its own virtual CPU/memory, one program's misbehavior is far less likely to corrupt another's.
- **Resource utilization** — many more programs can "run" than there are literal physical CPUs or literal physical memory bytes, because most programs aren't using their full share at every single instant.

## Disadvantages of Virtualization

- **Overhead** — maintaining the illusion (context switches, address translation on every memory access) costs real CPU cycles, some of which hardware assistance (like a TLB, Module 09) exists specifically to claw back.
- **Leaky abstraction under extreme load** — if too many programs genuinely need CPU or memory at once, the illusion of "you have plenty" strains, and programs experience real slowdowns (scheduling delays, page faults, thrashing — Module 11) despite the illusion trying to hide it.

## Best Practices

- Whenever you meet a new OS mechanism, ask first: "is this virtualizing the CPU, or virtualizing memory?" Nearly everything in Modules 02–11 sorts cleanly into one of those two buckets.
- Keep the **mechanism vs. policy** split in mind for both: "how do we physically switch/translate" (mechanism) is always a separate question from "what's the best choice given the options" (policy) — Module 03 vs. Modules 04–05 for the CPU is the clearest example.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Virtualization means each program actually gets its own CPU core." | It means each program is convincingly *told* it does. Physically, many programs typically share far fewer real cores than the number of running programs. |
| "Virtual memory is the same thing as swap space / disk-backed memory." | Virtual memory is the *abstraction* every program's addresses live in — it exists even on a machine with no swap space at all. Swapping to disk (Module 10) is one specific mechanism used to *extend* that abstraction beyond physical RAM's size, not a definition of it. |

## Interview Questions

1. **Q: What does it mean to "virtualize" the CPU?**
   A: To give every running program the illusion of having its own dedicated, continuously-running CPU, while in reality the OS rapidly switches a small number of physical CPUs between many programs.

2. **Q: What's the difference between virtualizing the CPU and virtualizing memory?**
   A: CPU virtualization is about *time* — sharing one physical resource across programs by giving each a turn. Memory virtualization is about *space* — giving each program its own private range of addresses that get translated to some (possibly scattered, possibly partially non-resident) location in physical RAM.

3. **Q: Why doesn't every program just get a real, dedicated CPU core and a real, dedicated chunk of RAM instead of a virtualized illusion?**
   A: There typically aren't enough physical cores or enough physical RAM to give every simultaneously running program (often hundreds) a literal dedicated share — and even if there were, programs would need to be written to explicitly coordinate over CPU time and physical addresses with every other program, which virtualization eliminates entirely.

## Summary

- Virtualization takes one scarce physical resource and presents each program with its own private, virtual version of it.
- The CPU is virtualized via mechanism (limited direct execution, Module 03) plus policy (scheduling, Modules 04–05).
- Memory is virtualized via the address space abstraction (Module 06), address translation (Modules 07–09), and mechanisms for exceeding physical RAM's actual size (Modules 10–11).
- The next topic, Concurrency, covers what goes wrong the moment virtualization succeeds well enough that truly multiple things are happening "at once."
