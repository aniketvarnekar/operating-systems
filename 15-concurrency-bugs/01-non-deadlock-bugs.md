# Non-Deadlock Bugs

## Learning Objectives

By the end of this section you should be able to:
- Define an atomicity violation and give a concrete example
- Define an order violation and give a concrete example
- Explain why these bugs are, in practice, at least as common as deadlock, despite receiving less attention in introductory treatments

## Prerequisites

- Module 13 in full
- Module 14, Topic 1 (Condition Variables and the Producer-Consumer Problem)

## Motivation

Deadlock (Module 14, Topic 3; formalized in Topics 2–3 of this module) tends to get the most attention in OS courses because it's dramatic and conceptually clean — but real-world bug studies of concurrent software consistently find that a majority of actual concurrency bugs are not deadlocks at all. This topic covers the two other major categories, which are subtler precisely because nothing "hangs" — the program just quietly produces a wrong result.

## Problem Statement

Not every concurrency bug involves threads waiting on each other in a cycle. Sometimes a program simply forgets to protect a piece of shared data that needs protecting, or assumes one thread's operation will happen before another's without actually enforcing that order. What do these mistakes look like concretely, and how do they differ from each other?

## Concept

### Atomicity Violations

> An **atomicity violation** occurs when a sequence of memory accesses (or operations) that was intended to execute as a single, indivisible unit is instead interrupted by another thread's access, because the necessary synchronization (a lock, Module 13) was missing or incomplete.

This is precisely Module 01, Topic 4's lost-update example, generalized: any code that assumes "check, then act" (or any other multi-step sequence involving shared data) happens without interruption, but fails to actually enforce that with a lock, is vulnerable to an atomicity violation. A common, subtle real-world variant: a program correctly locks *most* accesses to a shared variable, but has one code path — perhaps a rarely-executed error-handling branch — that reads or writes the same variable **without** acquiring the lock first. The bug is invisible during normal testing (the common paths are all correctly locked) and appears only when that one unprotected path happens to run concurrently with a locked one.

```c
// Thread 1                          // Thread 2
if (thread_ptr != NULL) {             thread_ptr = NULL;   // NOT locked!
    lock(&mutex);                     // races directly against
    thread_ptr->data++;                // Thread 1's check-then-use
    unlock(&mutex);
}
```

Here, Thread 1's check (`thread_ptr != NULL`) and its subsequent use are not protected as one atomic unit against Thread 2's unprotected write — Thread 2 can set `thread_ptr` to NULL in the gap between Thread 1's check and its use, causing Thread 1 to dereference a NULL pointer despite the check having just passed.

### Order Violations

> An **order violation** occurs when code assumes a particular ordering between two threads' operations (e.g., "thread A's initialization always happens before thread B tries to use the result"), but nothing actually enforces that ordering — leaving it to depend on however the scheduler (Modules 04–05) happens to interleave the two threads, exactly as Module 12, Topic 3 warned.

```c
// Thread 1 (creator)                 // Thread 2 (spawned)
thread_create(&t2, my_thread, ...);   void *my_thread(void *arg) {
mydata = 42;                              print(mydata);   // assumes mydata
                                            // is already 42 — NOT guaranteed!
                                        }
```

Here, there's no guarantee that Thread 1's assignment `mydata = 42` executes before Thread 2's `print(mydata)` — Module 12, Topic 3 already established that a newly-created thread's execution isn't ordered relative to the creating thread's continued execution without explicit synchronization (like a condition variable, Module 14, Topic 1, specifically used to enforce "wait until mydata has actually been set").

### Why These Bugs Are Common and Dangerous

Both categories share a key property with race conditions generally (Module 01, Topic 4): they are **timing-dependent**. An atomicity violation might never manifest during testing if the unprotected code path is rarely exercised, or if the specific interleaving needed to trigger it happens to not occur on the testing machine's particular timing. An order violation might work correctly every single time on a fast machine (where Thread 1 happens to always finish first, purely by chance) and fail unpredictably on a slower or more heavily-loaded machine, or under different scheduling conditions (Modules 04–05). Neither category requires a waiting cycle (unlike deadlock) — the program doesn't hang, it just silently produces a wrong (or crashing) result some of the time.

## Internal Working (Preview)

```
 Atomicity violation:

   Thread 1: [check ptr != NULL] ─────► [use ptr->data]
   Thread 2:          [ptr = NULL, UNPROTECTED] ──────────┘
                        ▲
              lands in the gap between Thread 1's
              check and use — NOT prevented by any lock,
              since this specific access was never locked


 Order violation:

   Thread 1 (creator):  create(Thread 2) ──► mydata = 42
   Thread 2 (spawned):        print(mydata)  ◄── runs BEFORE
                                                   mydata = 42 executes —
                                                   NO enforced ordering
                                                   guarantees otherwise
```

## Real-World Analogy

An **atomicity violation** is like a shared office whiteboard where everyone has agreed to only erase and rewrite the schedule while holding a designated marker (the lock) — except one person, in a hurry, sometimes grabs a different marker and edits it directly without checking whether anyone's mid-edit with the official one, causing occasional half-overwritten, garbled schedules that only appear when that shortcut happens to coincide with someone else's edit. An **order violation** is like assuming a new employee will always read the onboarding handbook (an initialization step) before their first day of actual work (using the result) — without ever explicitly confirming they've finished reading it first, so on a day when HR is running behind, the employee might show up and start working with no idea what's actually expected of them yet.

## Why These Bugs Deserve Explicit Attention

Because deadlock is conceptually clean and dramatic (the program visibly hangs), it's easy to over-focus on it as "the" concurrency bug to watch for. But atomicity and order violations don't hang — they silently corrupt data or crash unpredictably, often much harder to notice, reproduce, and diagnose precisely because there's no obvious symptom (like a frozen program) pointing directly at the cause. Recognizing these as a distinct, common bug category — not just "an incomplete deadlock" — is essential for effective real-world debugging of concurrent code.

## Advantages of Understanding This Distinction

- **Sharper debugging instincts** — a program that silently produces wrong results (rather than hanging) should immediately suggest atomicity or order violations, not deadlock, as the likely cause.
- **More targeted code review** — knowing to look specifically for "is every access to this shared variable consistently protected by the same lock" (atomicity) and "does this code assume an ordering that isn't actually enforced" (order) gives concrete, checkable review criteria.

## Disadvantages / Real Dangers

- **Extremely difficult to reproduce reliably** — like all race conditions (Module 01, Topic 4), the specific timing needed to trigger these bugs may occur rarely, making them notorious for passing all local testing and appearing only in production, under different load or hardware conditions.
- **No single symptom pinpoints them** — unlike deadlock's visible hang, these bugs can manifest as anything from a wrong computed value to a crash, making root-cause diagnosis genuinely harder.

## Best Practices

- Ensure **every** access to a piece of shared data is protected by the **same** lock, consistently, across **every** code path — including rare error-handling branches — not just the common-case paths that get exercised most often during testing.
- Never assume an ordering between two threads' operations unless it's explicitly enforced via a condition variable (Module 14, Topic 1), a semaphore (Module 14, Topic 2), or an equivalent synchronization mechanism — "it usually works in that order" is not a guarantee.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "If a concurrent program isn't deadlocking (hanging), it's free of concurrency bugs." | Atomicity and order violations don't cause hangs at all — they cause silent data corruption or unpredictable crashes, and real-world studies find them to be at least as common as deadlock in practice. |
| "As long as most accesses to a shared variable are locked, the code is safe." | Even a single unprotected access path (e.g., a rare error-handling branch) can race against the correctly-locked paths, causing an atomicity violation that only manifests when that rare path happens to execute concurrently with a locked one. |

## Interview Questions

1. **Q: What is an atomicity violation?**
   A: A sequence of memory accesses intended to execute as one indivisible unit is instead interrupted by another thread's access, because the necessary lock was missing or incompletely applied — often because one code path (e.g., a rare error branch) doesn't acquire the same lock as the rest.

2. **Q: What is an order violation?**
   A: Code assumes a particular execution order between two threads' operations without actually enforcing it via synchronization, leaving the actual order to depend on however the scheduler happens to interleave them — which is never guaranteed by default (Module 12, Topic 3).

3. **Q: Why are non-deadlock concurrency bugs often considered more dangerous in practice than deadlock, despite receiving less introductory attention?**
   A: They don't produce an obvious symptom like a hang — they cause silent data corruption or unpredictable crashes that are timing-dependent and hard to reproduce, making root-cause diagnosis significantly harder than for deadlock's visible freeze.

## Summary

- Atomicity violations occur when a sequence of shared-data accesses meant to be indivisible is interrupted by another thread's unsynchronized access — often via one inconsistently-locked code path.
- Order violations occur when code assumes an execution order between threads that isn't actually enforced by any synchronization mechanism.
- Both are timing-dependent, silent (no hang), and at least as common in practice as deadlock, making them essential to recognize as a distinct bug category.
- The next topic formalizes deadlock itself — the four necessary conditions for it to occur, and prevention strategies that each attack one of those conditions directly.
