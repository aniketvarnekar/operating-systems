# The Basic Idea of Locks

## Learning Objectives

By the end of this section you should be able to:
- Define a critical section and mutual exclusion precisely
- Explain the lock/unlock interface and how it wraps a critical section
- Connect this directly back to Module 01, Topic 4's lost-update example

## Prerequisites

- Module 01, Topic 4 (Concurrency)
- Module 12, Topic 3 (The Thread API and Creation Patterns)

## Motivation

Module 01, Topic 4 showed a concrete lost-update bug: two threads interleaving a read-modify-write sequence on a shared counter, silently losing one increment. Module 12 gave you the vocabulary (threads sharing an address space) and showed that interleaving is never guaranteed. This topic introduces the first, most fundamental tool for making that interleaving harmless: the lock.

## Problem Statement

Recall Module 01, Topic 4's example: `counter = counter + 1` compiles to a read, an add, and a write — three separate steps. If two threads can interleave these three steps arbitrarily (Module 12, Topic 3 established that no ordering is guaranteed), how can a program guarantee that this sequence of three steps behaves as one indivisible unit, regardless of how the scheduler happens to interleave the two threads?

## Concept

### The Critical Section

> A **critical section** is a region of code that accesses shared data (or a shared resource) in a way that must not be interleaved with another thread's access to that same data — it must, from the perspective of correctness, execute as if it were one indivisible unit.

`counter = counter + 1`, in Module 01, Topic 4's example, is a critical section: its three underlying steps must not be interleaved with another thread's three steps operating on the same `counter` variable.

### Mutual Exclusion

> **Mutual exclusion** is the guarantee that, at any given moment, at most one thread is executing within a given critical section — every other thread that wants to enter must wait until the current occupant leaves.

This is precisely the property that makes a critical section behave as if it were indivisible: as long as no two threads are ever inside it simultaneously, the specific interleaving of *other* code around it doesn't matter — the critical section's own internal steps can never be split apart by another thread's competing access to the same data.

### The Lock/Unlock Interface

> A **lock** is a variable (with just two states, roughly "held" or "free") that a thread uses to enforce mutual exclusion: a thread calls **lock()** before entering a critical section, and **unlock()** after leaving it. If a thread calls lock() while another thread already holds it, the calling thread is made to wait until the lock becomes free.

```c
lock(&mutex);
counter = counter + 1;   // critical section — protected
unlock(&mutex);
```

With this in place, revisit Module 01, Topic 4's interleaving: Thread A calls `lock()`, acquires it, and only *then* reads, adds, and writes `counter` — Thread B, attempting `lock()` at any point before A calls `unlock()`, is forced to wait. The dangerous interleaving (B reading `counter` before A's write lands) becomes structurally impossible, since B cannot even begin its own read until A has completed its entire critical section and released the lock.

## Internal Working (Preview)

```
 WITHOUT a lock (Module 01, Topic 4's bug):

   Thread A:  read counter(10) ──────► add 1 ──► write counter(11)
   Thread B:         read counter(10) ──────► add 1 ──► write(11)
                             ▲
                     B reads BEFORE A's write — lost update


 WITH a lock:

   Thread A:  lock() ──► read(10) ──► add 1 ──► write(11) ──► unlock()
   Thread B:  lock() ──► [BLOCKED until A calls unlock()] ──► read(11) ──► add 1 ──► write(12)
                                                                 ▲
                                            B can only proceed AFTER A's full
                                            critical section completes — correct!
```

## Real-World Analogy

Think of a single-occupancy restroom with a lockable door. The lock doesn't prevent multiple people from *wanting* to use it at the same time — it prevents more than one person from actually being *inside* at the same time. Whoever gets there first locks the door (acquires the lock); anyone else who arrives simply waits outside (blocked) until the door unlocks, no matter how the arrival order between different people actually played out — the "at most one occupant" guarantee is what matters, not the arrival order itself.

## Why This Design Is Necessary

Module 01, Topic 4 established that ordinary sequential-looking code (`counter = counter + 1`) is not atomic at the hardware level, and that threads' relative execution order is never guaranteed (Module 12, Topic 3). A lock is the minimal, general-purpose tool that converts "this sequence of steps must not be interleaved with another thread's matching sequence" into an enforced guarantee, regardless of how the scheduler happens to interleave everything else.

## Advantages of the Lock/Unlock Interface

- **Directly, precisely solves the lost-update problem** from Module 01, Topic 4, by making the critical section behave as an indivisible unit.
- **Simple, general-purpose interface** — the same two calls (lock/unlock) apply to protecting any shared data, regardless of what that data actually is.

## Disadvantages / Costs

- **Serializes access** — by design, only one thread may be inside the critical section at a time, which limits how much genuine parallelism is possible for code that touches the protected data, even across many CPU cores.
- **Introduces new risks of its own** — forgetting to unlock, or acquiring multiple locks in an inconsistent order across different threads, opens the door to deadlock, covered in Module 15.
- **How a lock is actually implemented under the hood is not yet obvious** — the next topic shows why implementing lock() and unlock() correctly is harder than it first appears.

## Best Practices

- Keep critical sections as short as possible — only the specific operations that genuinely need mutual exclusion should be inside the lock/unlock pair; unnecessarily long critical sections needlessly serialize unrelated work.
- Always pair every lock() with a corresponding unlock(), on every code path, including error/exception paths — an unreleased lock permanently blocks every other thread waiting on it.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A lock prevents other threads from running at all while one thread holds it." | A lock only prevents other threads from entering that *specific* critical section (or any code section protected by that same lock) — other threads remain free to run any other code that doesn't require that lock. |
| "Locks are only needed for extremely long or complex operations." | Even a single, simple-looking line like `counter = counter + 1` is not atomic at the hardware level (Module 01, Topic 4) — locks are needed for any shared-data access with multiple underlying steps, however small it looks in source code. |

## Interview Questions

1. **Q: What is a critical section, and what does mutual exclusion guarantee about it?**
   A: A critical section is a region of code accessing shared data that must not be interleaved with another thread's access to the same data. Mutual exclusion guarantees that at most one thread executes within a given critical section at any moment, forcing any other thread to wait.

2. **Q: How does a lock solve the lost-update problem from Module 01, Topic 4?**
   A: By requiring a thread to acquire the lock before entering the critical section (e.g., the read-modify-write on a shared counter) and release it afterward, any other thread attempting the same critical section is forced to wait until the current thread's entire sequence completes — making the dangerous interleaving structurally impossible.

3. **Q: What's the trade-off introduced by using a lock?**
   A: It serializes access to the protected critical section — only one thread may execute it at a time, limiting genuine parallelism for that specific piece of code, even on a multi-core system with capacity for more.

## Summary

- A critical section is code accessing shared data that must not be interleaved with another thread's matching access; mutual exclusion guarantees at most one thread is inside it at a time.
- A lock enforces mutual exclusion via a simple lock()/unlock() interface, directly solving the lost-update race condition from Module 01, Topic 4.
- The cost is serialized access to the protected section, and new risks (like deadlock, Module 15) that come with using locks incorrectly.
- The next topic addresses a question this one deliberately left open: how is lock() itself actually implemented correctly, given that its own internal logic faces exactly the same atomicity problem it's meant to solve?
