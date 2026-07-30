# Module 01 — Introduction to Operating Systems

## Module Goal

By the end of this module, you will understand **what an operating system actually is, why it exists, and the three big ideas that every later module is really just a deep dive into** — before you look at a single line of process, memory, or file-system code. Every later module builds on this mental model. Skipping it is the #1 reason people memorize scattered OS facts (deadlock, page tables, journaling) without ever seeing that they're all the same handful of tricks applied to different resources.

## Topics Covered in This Module

1. **[What Is an Operating System?](01-what-is-an-operating-system.md)** — The precise role of an OS: a resource manager and a "standard library" for hardware, sitting between raw hardware and every user program.
2. **[History of Operating Systems](02-history-of-operating-systems.md)** — Where OS design came from (batch processing, multiprogramming, time-sharing), and why today's OS looks the way it does because of decades of hardware-driven pressure.
3. **[Virtualization](03-virtualization.md)** — The first big idea: turning one scarce physical CPU and one physical memory into the illusion of many private, abundant ones.
4. **[Concurrency](04-concurrency.md)** — The second big idea: the problems that appear the moment more than one thing can happen "at the same time," and why they're subtler than they look.
5. **[Persistence](05-persistence.md)** — The third big idea: how data is made to outlive the process that created it — and survive a power loss or crash mid-write.
6. **[System Calls and the OS Interface](06-system-calls-and-the-os-interface.md)** — The actual mechanism a user program uses to ask the OS for help, and why this boundary is enforced by hardware, not just convention.
7. **[Module Summary](07-module-summary.md)** — Consolidated recap tying all three big ideas together.

## Prerequisites

- General programming literacy: you should be comfortable writing and running programs in *some* language, and know what a function call, a variable, and a running program are.
- No prior operating systems knowledge required.
- No specific software required for this module — it's entirely conceptual.

## How to Study This Module

Read in order. Topics 1–2 build the "why does this field exist at all" foundation. Topics 3–5 are the most important topics in the entire course — they introduce virtualization, concurrency, and persistence as named ideas for the first time, and literally every module from Module 02 onward is a deep dive into one of these three. Read them slowly; a rough mental model now will make every later module feel like "oh, this is just that same idea again," instead of a pile of unrelated new facts. Topic 6 gives you the concrete mechanism (system calls) that ties the abstract ideas to real, callable operations.
