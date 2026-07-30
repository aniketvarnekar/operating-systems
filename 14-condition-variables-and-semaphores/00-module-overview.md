# Module 14 — Condition Variables and Semaphores

## Module Goal

By the end of this module, you will understand **the tools for thread coordination that go beyond simple mutual exclusion** — condition variables, which let a thread wait for a specific condition to become true, and semaphores, a single primitive general enough to express both mutual exclusion and condition-based waiting.

## Topics Covered in This Module

1. **[Condition Variables and the Producer-Consumer Problem](01-condition-variables-and-the-producer-consumer-problem.md)** — Why a lock alone can't express "wait until this becomes true," and how condition variables solve it, motivated by the classic bounded-buffer problem.
2. **[Semaphores as a Unifying Primitive](02-semaphores-as-a-unifying-primitive.md)** — A single counting primitive that can implement both locks and condition-variable-style waiting.
3. **[Classic Concurrency Problems](03-classic-concurrency-problems.md)** — The dining philosophers problem, and what it teaches about deadlock-safe resource acquisition.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 13 in full — this module assumes fluency with locks and the mutual-exclusion goal they solve.

## How to Study This Module

Read in order. Topic 1 introduces a problem locks alone cannot solve — waiting for a condition, not just waiting for exclusive access — using the bounded-buffer producer-consumer problem as a concrete, motivating example. Topic 2 introduces semaphores as a more general, single primitive that can express everything from Topic 1 (and Module 13) using one consistent interface. Topic 3 applies everything so far to the dining philosophers problem, a classic scenario specifically designed to expose resource-ordering dangers — direct preparation for Module 15's deadlock coverage.
