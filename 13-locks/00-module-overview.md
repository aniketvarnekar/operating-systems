# Module 13 — Locks

## Module Goal

By the end of this module, you will understand **locks — the fundamental tool for protecting shared data from the race conditions Module 12 set up** — including the hardware instructions locks are built from, and the trade-off between spinning and sleeping while waiting for one.

## Topics Covered in This Module

1. **[The Basic Idea of Locks](01-the-basic-idea-of-locks.md)** — Critical sections, mutual exclusion, and the lock/unlock interface.
2. **[Building Locks: Hardware Support](02-building-locks-hardware-support.md)** — Why ordinary load/store instructions can't implement a correct lock, and the hardware primitives (test-and-set, compare-and-swap) that can.
3. **[Spin Locks vs. Two-Phase Locks](03-spin-locks-vs-two-phase-locks.md)** — The cost of waiting by spinning vs. sleeping, and the hybrid, futex-based approach real systems use.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 12 in full, especially Topic 2 (The Thread Abstraction) and Topic 3's worked example of unpredictable interleaving.
- Module 01, Topic 4 (Concurrency) — the race-condition example this module directly protects against.

## How to Study This Module

Read in order. Topic 1 defines the goal (mutual exclusion over a critical section) in the abstract. Topic 2 is the crux of the module: it shows, concretely, why this goal cannot be achieved using only ordinary instructions, and introduces the specific hardware support that makes it possible at all. Topic 3 then covers the practical engineering question of what a thread should actually *do* while waiting for a lock it can't yet acquire — a question with a real, measurable performance answer.
