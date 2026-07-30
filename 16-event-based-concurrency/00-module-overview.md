# Module 16 — Event-Based Concurrency

## Module Goal

By the end of this module, you will understand **event-based concurrency — handling many concurrent tasks using a single thread and an event loop, instead of one OS thread per task** — including why it avoids Module 15's entire concurrency-bug category for shared in-memory state, and the specific problem (blocking calls) it must solve to work at all.

## Topics Covered in This Module

1. **[The Event Loop and select()/poll()](01-the-event-loop-and-select-poll.md)** — Handling many I/O sources with a single thread, checking readiness rather than dedicating a thread per source.
2. **[The Blocking Call Problem and epoll](02-the-blocking-call-problem-and-epoll.md)** — Why a single blocking call anywhere in an event loop defeats the entire model, and epoll's improvement over select/poll's scaling limits.
3. **[Module Summary](03-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 12 in full — event-based concurrency is best understood by direct contrast with the thread-per-task model that module established.
- Module 04, Topic 6 (Incorporating I/O) — the CPU/I/O burst model this module directly extends to a single-threaded context.

## How to Study This Module

Read in order. Topic 1 introduces the core mechanism — a loop that asks the OS "which of these many things I care about is ready right now?" rather than dedicating a whole thread to waiting on each one individually. Topic 2 covers the single sharpest limitation of this model (a blocking call anywhere freezes everything) and the practical, scalable answer (epoll) real high-performance servers use.
