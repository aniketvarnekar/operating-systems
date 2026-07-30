# Concurrency

## Learning Objectives

By the end of this section you should be able to:
- Define concurrency and distinguish it from virtualization
- Explain, with a concrete example, why "multiple things happening at once" creates correctness problems that don't exist when only one thing happens at a time
- Identify which later modules are direct deep dives into concurrency

## Prerequisites

- Topic 3 (Virtualization)

## Motivation

Concurrency bugs are famous for being some of the hardest bugs in all of software to find and fix — they can appear only once in a million runs, vanish the moment you add a debug print statement, and pass every test on your machine while failing in production. Understanding *why* concurrency is fundamentally harder than sequential code — before you ever write a `lock()` call — is what makes Modules 12–16 make sense instead of feeling like arbitrary rules ("always lock this," "never do that") to memorize.

## Problem Statement

Suppose two independent parts of a program both want to increment the same shared counter variable, at what is effectively the same moment. Each increment looks like a single, atomic step in source code:

```
counter = counter + 1
```

But on real hardware, this single line is actually at least three separate steps:
1. **Read** the current value of `counter` from memory into a register.
2. **Add** one to that register's value.
3. **Write** the register's value back to `counter` in memory.

If two threads interleave these three steps — Thread A reads `counter` (say, it's 10), then Thread B *also* reads `counter` (still 10, since A hasn't written yet), then A writes back 11, then B *also* writes back 11 — the counter ends up at 11, even though two full increments happened. One increment was silently lost, with no error, no crash, and no obvious symptom — just a quietly wrong final number.

This is a **race condition**: the correctness of the result depends on the unpredictable, moment-to-moment timing of two things happening "at once," rather than depending only on the logic of the code itself.

## Concept

### Definition

> **Concurrency**, in an OS context, refers to the set of problems (and the primitives that solve them) that arise specifically because multiple threads of execution can be active — genuinely in parallel on multiple CPU cores, or interleaved rapidly on one core — and may access shared data or resources at unpredictable, overlapping moments.

Concurrency is the direct *consequence* of virtualization working well: because the OS can run so many threads "at once" (Topic 3), those threads inevitably sometimes need to touch the same shared data — and that's exactly where correctness gets hard.

### Why This Is Fundamentally Different From Sequential Bugs

A bug in purely sequential code is deterministic: given the same input, it fails the same way, every time, and you can reliably reproduce it by just re-running the program. A **race condition** is often *not* deterministic in this way — whether it manifests depends on the precise, sub-microsecond timing of how two threads' instructions happen to interleave on a given run, which can be influenced by CPU load, other running programs, or even whether a debugger is attached. This is exactly why concurrency bugs have a reputation for being uniquely difficult: the bug can be real, present, and dangerous in production, yet refuse to show up reliably (or at all) in your local testing.

### The General Shape of the Solution

Every concurrency primitive in Modules 12–16 exists to solve some version of the same underlying need: **make a sequence of operations on shared data behave as if it happened as one indivisible, uninterruptible step**, even though it's actually built from multiple hardware instructions.

- **Locks** (Module 13) let a thread declare "no one else may touch this shared data until I say I'm done."
- **Condition variables and semaphores** (Module 14) let threads coordinate not just around "who's touching shared data," but around waiting for a specific condition to become true (e.g., "wait until there's an item in the queue").
- **Deadlock and other concurrency bugs** (Module 15) study the specific, dangerous mistakes that arise when these coordination tools are used incorrectly.

## Internal Working (Preview)

```
 Thread A:  read counter (10) ──────────► add 1 ──► write counter (11)
 Thread B:         read counter (10) ──────────► add 1 ──► write counter (11)
                          ▲
                          └── B reads BEFORE A's write lands — the lost update.

 Expected final value: 12 (two increments of 10)
 Actual final value:   11 (one increment silently lost)
```

## Real-World Analogy

Imagine two people sharing one bank account, both looking at a paper balance sheet at the same moment. Person A reads "$100," walks away to add their $10 deposit slip. Person B, before A returns, *also* reads "$100" (A hasn't written the new number yet), and walks away to add their own $20 deposit slip. Whoever writes their new total back to the sheet *last* simply overwrites the other's update — the balance sheet ends up short by one of the two deposits, with no sign anything went wrong. This is exactly a lost update — the paper-based system has no built-in mechanism forcing "read, calculate, write" to happen as one uninterrupted unit per person.

## Why Concurrency Requires Deliberate Design

The core issue is that ordinary source-code lines (like `counter = counter + 1`) give a false impression of atomicity — they *look* like one step, but the hardware executes them as several. Any time more than one thread can interleave its own multi-step operations with another thread's multi-step operations on the *same* shared data, correctness stops being guaranteed by the code's logic alone and starts depending on timing — which is precisely the class of problem concurrency primitives are built to eliminate, deterministically, regardless of how threads happen to interleave.

## Advantages of Embracing Concurrency (Done Correctly)

- **Performance** — genuinely parallel work across multiple CPU cores can finish dramatically faster than doing the same work on one core sequentially.
- **Responsiveness** — a program (e.g., a UI) can keep responding to input on one thread while a slow operation (a network request, a large computation) runs on another.

## Disadvantages / Real Dangers

- **Race conditions** — silent, timing-dependent data corruption, as shown above.
- **Deadlock** — two or more threads permanently stuck waiting on each other, covered in depth in Module 15.
- **Reproducibility** — concurrency bugs can be extraordinarily hard to reliably reproduce and debug, precisely because they depend on timing rather than pure logic.

## Best Practices

- Treat "is this data shared across more than one thread?" as the first question for any variable — if the answer is yes and it can be modified, assume it needs explicit coordination (Modules 13–14) until proven otherwise.
- Never assume "this race window is too small to matter in practice" — race conditions that occur one time in a million runs are still guaranteed, eventually, to occur in a long-running production system.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "If my code passes all my local tests, it doesn't have a race condition." | Race conditions are timing-dependent; a specific interleaving that causes corruption might never occur on your machine's specific hardware/load conditions but occur regularly on a busier production server. |
| "A single line of code like `x++` is always a single, atomic operation." | It typically compiles to multiple separate machine instructions (read, modify, write), any of which can be interleaved with another thread's instructions on the same variable. |

## Interview Questions

1. **Q: What is a race condition?**
   A: A situation where the correctness of a program's outcome depends on the unpredictable relative timing/interleaving of multiple threads accessing shared data, rather than depending only on the program's logic — often causing an update to be silently lost or corrupted.

2. **Q: Why are concurrency bugs often harder to debug than sequential bugs?**
   A: Because their manifestation depends on precise timing/interleaving rather than deterministic logic — the same buggy code can pass every test on one machine/run and fail unpredictably under different timing conditions (e.g., higher production load), making them hard to reliably reproduce.

3. **Q: Why does `counter = counter + 1` need protection in a multithreaded program, even though it looks like one line of code?**
   A: Because it actually compiles to multiple separate hardware steps (read, add, write); if another thread's own read/add/write for the same variable interleaves between those steps, one thread's update can be silently overwritten/lost.

## Summary

- Concurrency is the set of correctness problems (and solving primitives) that appear once multiple threads can access shared data at unpredictable, overlapping times.
- Concurrency bugs, especially race conditions, are timing-dependent and notoriously hard to reproduce and debug.
- Every concurrency primitive (locks, condition variables, semaphores) exists to make a multi-step operation on shared data behave as one indivisible step, regardless of thread interleaving.
- The next topic, Persistence, covers the third big idea: making data survive not just concurrent access, but the process ending entirely — or the machine losing power mid-write.
