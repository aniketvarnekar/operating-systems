# Module 06 — Memory API and Address Spaces

## Module Goal

By the end of this module, you will understand **the address space abstraction — the memory equivalent of the process abstraction from Module 02 — and the concrete API real programs use to request and release memory dynamically**. This module begins Virtualization's second half (memory), setting up the goals every mechanism in Modules 07–11 exists to fulfill.

## Topics Covered in This Module

1. **[The Address Space Abstraction](01-the-address-space-abstraction.md)** — What an address space is, why every process needs its own private one, and the three explicit goals (transparency, efficiency, protection) it must satisfy.
2. **[The Memory API](02-the-memory-api.md)** — The malloc/free family real C programs use to request and release heap memory, and what actually happens underneath each call.
3. **[Common Memory Bugs](03-common-memory-bugs.md)** — Memory leaks, dangling pointers, buffer overflows, and other classic mistakes manual memory management makes possible.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 01, Topic 3 (Virtualization)
- Module 02, Topic 1 (The Process Abstraction)

## How to Study This Module

Read in order. Topic 1 is conceptual and foundational — the three goals it introduces (transparency, efficiency, protection) are the yardstick every memory-virtualization mechanism in Modules 07–11 will be judged against, so keep them in mind as you move forward. Topic 2 gets concrete and hands-on: the actual function calls a C program uses to manage memory at runtime. Topic 3 is deliberately cautionary — manual memory management is one of the most bug-prone areas of systems programming, and seeing the classic failure modes clearly here will make you a sharper reader of code (in any language, not just C) for the rest of your career.
