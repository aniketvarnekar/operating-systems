# Module 12 — Concurrency: Threads

## Module Goal

By the end of this module, you will understand **the thread abstraction — multiple independent sequences of execution sharing a single address space — and the API real programs use to create and manage them**. This module begins the course's second major theme (Concurrency, introduced conceptually in Module 01, Topic 4), setting up the specific vocabulary and mental model that Modules 13–16 will build a full concurrency toolkit on top of.

## Topics Covered in This Module

1. **[Why Threads?](01-why-threads.md)** — The motivation for running multiple sequences of execution within one process, and how a thread differs from a process.
2. **[The Thread Abstraction](02-the-thread-abstraction.md)** — What's shared between threads in the same process, what's private to each, and the per-thread bookkeeping the OS must maintain.
3. **[The Thread API and Creation Patterns](03-the-thread-api-and-creation-patterns.md)** — The concrete function calls (create, join) real programs use, and common patterns for structuring multithreaded code.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 01, Topic 4 (Concurrency) — the conceptual introduction to race conditions this module builds directly on.
- Module 02 in full, especially Topic 1 (The Process Abstraction) and Topic 3 (The Process Control Block).

## How to Study This Module

Read in order. Topic 1 answers "why would you want this at all" before any mechanism is introduced — motivation first, exactly as every module in this course tries to do. Topic 2 is the conceptual core: understanding precisely what threads within one process share (the address space) versus what remains private to each (registers, stack) is the single most important idea for making sense of every concurrency bug in Module 15. Topic 3 gets hands-on with the actual API — by the end, you should be able to both write and mentally trace through a simple multithreaded program.
