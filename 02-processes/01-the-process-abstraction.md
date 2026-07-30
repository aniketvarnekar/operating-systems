# The Process Abstraction

## Learning Objectives

By the end of this section you should be able to:
- Give a precise definition of a process
- Explain the difference between a program (on disk) and a process (running)
- List the pieces of machine state a process abstraction must account for

## Prerequisites

- Module 01, Topic 3 (Virtualization)

## Motivation

"Process" is one of the most overloaded, casually-used words in computing — people say "kill that process," "the process is using 80% CPU," "spawn a new process," often without ever pinning down exactly what the word refers to. Getting a precise, mechanical definition now makes every following topic in this module (states, the PCB, fork/exec/wait) click into place as details of one clear concept, instead of loosely related trivia.

## Problem Statement

A program on your disk — say, a compiled `sort` executable — is just a static, inert file: a sequence of bytes containing machine instructions and initial data. Sitting on disk, it does nothing; it has no notion of "currently executing," no current position in its own instructions, no data it's actively working on.

The moment you run that program, something fundamentally different exists: instructions are actively being fetched and executed, a stack is growing and shrinking as functions are called and return, and memory is being read and written. If you ran the *same* `sort` executable twice at once, sorting two different files, you'd clearly want two independent things — each with its own current instruction, its own stack, its own memory — even though both started from the exact same on-disk program.

What is the OS abstraction that captures "one specific, currently-executing instance of a running program," distinct both from the static file on disk and from any other simultaneously-running instance of that same file?

## Concept

### Definition

> A **process** is the OS's abstraction for a running program — a single, independent instance of program execution, encompassing everything needed to describe its current state: its memory contents (Module 06), the value of its CPU registers (including the crucial program counter, tracking exactly which instruction executes next), and its record of open files and other OS-managed resources.

### Program vs. Process

This distinction is the single most important thing to internalize in this topic:

| | Program | Process |
|---|---|---|
| **What it is** | A static file of instructions + data, sitting on disk | A specific, running instance of that program, in memory, actively executing |
| **How many can exist** | One file | Zero, one, or many simultaneously-running instances of the same program |
| **Does it change over time?** | No — until you recompile/edit it | Constantly — its registers, memory, and stack change with every instruction executed |

Running the same `sort` executable twice, on two different input files, creates **two separate processes** from **one program** — each with its own private illusion of memory and CPU, as promised by virtualization (Module 01, Topic 3).

### The Machine State a Process Encompasses

To fully describe "everything about one specific running instance," the OS must track (at minimum):

- **Memory (the address space)**: the instructions being executed and the data the process reads and writes — covered in depth starting in Module 06.
- **Registers**: small, extremely fast storage locations inside the CPU itself. Of particular importance is the **program counter (PC)** (sometimes called the instruction pointer), which tracks exactly which instruction the process should execute next — the single piece of state that most directly captures "where is this process, right now, in its own execution."
- **A stack pointer and frame pointer**: tracking function call arguments, local variables, and return addresses, growing and shrinking as functions are called and return.
- **I/O information**: a list of files the process currently has open, and other OS-managed resources it holds.

## Internal Working (Preview)

```
  On disk (a program):                In memory (a process):
  ┌────────────────────┐              ┌─────────────────────────┐
  │  sort.exe           │   run  ──►   │  Address Space           │
  │  (instructions +    │              │  (code, data, heap,      │
  │   initial data,     │              │   stack — Module 06)     │
  │   inert bytes)      │              │  Registers (incl. PC)    │
  └────────────────────┘              │  Open file list          │
                                        └─────────────────────────┘
                                        one instance = one process
```

Run the same `sort.exe` twice concurrently on two different files, and you get **two** boxes on the right — same starting instructions, but two entirely separate address spaces, register sets, and program counters, each unaware of the other.

## Real-World Analogy

Think of a program like a movie script, and a process like an actual live performance of that script. The script itself (the program) is a static document — the same words on the page regardless of how many times it's performed. Each individual performance (a process) has its own actors currently at a specific line (the program counter), its own props and set currently in whatever state the ongoing performance has left them (memory), and its own list of currently-borrowed costumes (open files). You could stage the exact same script simultaneously in two different theaters — two independent performances (processes), both derived from the identical unchanging script (the program), each completely oblivious to how far along the other one currently is.

## Why the OS Needs This Abstraction

Without a clean process abstraction, the OS would have no clear unit to which it could apply memory protection, CPU scheduling, or resource accounting — "the CPU" and "the memory" are shared, singular physical resources, but "give this specific unit of work its own protected memory and its own fair share of CPU time" requires *something* to be the unit being protected and scheduled. The process is precisely that unit — it's the concrete, trackable "thing" that Module 01's virtualization promises are made *to*.

## Advantages of the Process Abstraction

- **Isolation** — each process gets its own private-feeling address space (Module 06), so one process's bugs don't directly corrupt another's memory.
- **A natural unit for scheduling and accounting** — the OS can fairly allocate CPU time (Modules 04–05) and track resource usage per process.
- **Multiple instances of the same program coexist cleanly** — running the same executable many times at once just creates many independent processes, with zero special-casing required.

## Disadvantages / Costs

- **Overhead of creation and management** — allocating a fresh address space, a process control block (Topic 3), and tracking all associated state costs real time and memory compared to a hypothetical "just jump to some code" alternative.
- **Communication between processes is deliberately harder** — because processes are isolated by design, sharing data between two processes requires explicit OS-mediated mechanisms (pipes, shared memory, sockets), unlike two functions within the same process that can simply share variables directly.

## Best Practices

- When you say "the process is doing X," make sure you can point to the specific machine state (its program counter's current position, its memory contents) that "doing X" corresponds to — this discipline prevents hand-wavy reasoning about program behavior.
- Always distinguish, out loud, between "the program" (the file) and "the process" (a specific running instance) — this precision alone eliminates a large fraction of beginner confusion about later topics like `fork()` (Topic 4).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A program and a process are basically the same thing, just different words for it." | A program is a static file; a process is a specific, live, running instance of executing that program, complete with changing register and memory state. One program can correspond to zero, one, or many simultaneous processes. |
| "If I run the same executable twice, it's still just 'one process' since it's the same code." | It creates two entirely separate processes, each with its own independent address space, registers, and program counter — they happen to share the same starting instructions, nothing more. |

## Interview Questions

1. **Q: What is a process?**
   A: The OS's abstraction for a running program — one specific, independent, executing instance, encompassing its memory (address space), CPU register values (including the program counter), and OS-managed resources like open files.

2. **Q: What's the difference between a program and a process?**
   A: A program is a static file of instructions and data sitting on disk. A process is a live, running instance of that program — its state (registers, memory contents, program counter) constantly changes as it executes, and multiple independent processes can run from the same single program.

3. **Q: If you run the same executable file three times simultaneously, how many processes exist, and do they share any state?**
   A: Three separate processes. By default they do not share memory or register state — each gets its own independent address space and execution context, even though all three started from the identical on-disk program.

## Summary

- A process is the OS's abstraction for one running instance of a program — its memory, its registers (especially the program counter), and its OS-managed resources.
- A program is a static file; a process is what exists once that program is actually running, and multiple independent processes can come from one program.
- The process is the fundamental unit the OS applies isolation, scheduling, and resource accounting to — it's the concrete "thing" CPU virtualization (Module 01, Topic 3) creates the illusion of privacy for.
- The next topic covers the specific states a process moves through during its lifetime, and what triggers each transition.
