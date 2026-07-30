# Spin Locks vs. Two-Phase Locks

## Learning Objectives

By the end of this section you should be able to:
- Explain what a spin lock is and when spinning is a reasonable strategy
- Explain the cost of spinning on a single-CPU system versus a multi-CPU system
- Describe the two-phase (futex-based) approach real systems use, and why it combines the best of both worlds

## Prerequisites

- Topic 2 (Building Locks: Hardware Support)
- Module 02, Topic 2 (Process States and Lifecycle) — specifically, the Blocked state

## Motivation

Topic 2 showed how to correctly implement lock() using test-and-set — but the naive implementation shown there has a thread that fails to acquire the lock simply loop, retrying immediately, forever, until it succeeds. This topic examines whether that's actually a good idea, and what real systems do instead.

## Problem Statement

Consider a thread that calls `lock()` on an already-held lock, using the test-and-set-based implementation from Topic 2. It enters a tight loop, repeatedly calling test-and-set as fast as possible, until the lock becomes free. What is this thread actually doing to the system while it waits — and is that the best possible use of its time?

## Concept

### Spin Locks

> A **spin lock** is a lock implementation where a thread waiting to acquire it repeatedly checks (via test-and-set or compare-and-swap, Topic 2) in a tight loop — "spinning" — consuming CPU cycles continuously until the lock becomes available.

### The Cost of Spinning Depends Heavily on CPU Count

- **On a single-CPU system**: a spinning thread is genuinely wasteful in a serious way. If Thread A holds the lock and is currently not running (because Thread B, which is spinning, was scheduled onto the one available CPU instead), Thread B's spinning does nothing but consume the CPU time that Thread A actually needs in order to finish its critical section and release the lock. In the worst case, a naive scheduler could let the spinning thread run for its entire time slice, accomplishing nothing, while the thread that could actually make progress (and release the lock) sits waiting in the Ready queue.
- **On a multi-CPU system**: spinning can be much more reasonable — if Thread A is actively running on a *different* CPU core and is expected to release the lock very soon, Thread B spinning (on its own, different core) briefly might cost less overall than the overhead of a full context switch (Module 03, Topic 4) to something else and back again.

This is a genuine, workload- and hardware-dependent trade-off, not a simple universal answer — spinning is potentially reasonable when the wait is expected to be very short and another CPU is actively making progress on releasing the lock; it's clearly wasteful when the lock-holder isn't even currently running.

### Two-Phase Locks: Combining Spinning and Sleeping

> A **two-phase lock** first spins for a short period (hoping the lock will become free quickly, cheaply avoiding a full context switch), and if it's still not free after that short spin, **puts the waiting thread to sleep** (moves it to the Blocked state, Module 02, Topic 2) instead of continuing to spin indefinitely — to be woken up later, specifically when the lock becomes available.

This is the practical, hybrid answer real systems (via a mechanism commonly called a **futex** — a "fast userspace mutex" — on Linux) actually use: an initial brief spin handles the common case cheaply (many critical sections are short, so the lock often becomes free almost immediately, making a full sleep/wake cycle's overhead unnecessary), while falling back to sleeping avoids the serious waste of spinning indefinitely on a lock that won't be released for a while.

### Why Sleeping Requires OS Involvement

Recall Module 02, Topic 2: moving a thread to Blocked, and later waking it back up to Ready once an event occurs, is an OS-level operation — it requires the scheduler (Modules 04–05) to stop considering this thread for CPU time, and later resume considering it, exactly the same mechanism used for a process blocked on I/O. A futex-based lock uses a system call (Module 01, Topic 6) specifically to hand this sleep/wake responsibility to the OS once the brief initial spin fails, rather than continuing to consume CPU time in user-mode spinning.

## Internal Working (Preview)

```
   Two-phase (futex-based) lock, from a waiting thread's perspective:

   attempt test_and_set() / compare_and_swap()  ── SUCCESS → done, lock acquired
              │
             FAIL
              │
              ▼
   spin briefly, retrying a few times   ── becomes free during this
   (cheap, avoids a full context switch)    brief window? → done
              │
        still not free after
        the brief spin
              │
              ▼
   SLEEP (move to Blocked, Module 02, Topic 2) —
   a system call hands this off to the OS scheduler,
   this thread consumes ZERO CPU while waiting
              │
              ▼
   WOKEN UP specifically when the lock becomes available
   (the unlocking thread signals waiters) → retry acquiring it
```

## Real-World Analogy

Think of waiting for a coworker to finish using a shared printer. If you expect them to be done any second (they're just clicking a final button), it might make sense to stand right there for a few seconds, checking (spinning) — faster than walking all the way back to your desk and getting a notification later. But if you check and they're clearly still in the middle of a large print job, continuing to stand there checking constantly wastes your own time uselessly — it makes far more sense to go back to your desk and do something else (sleep), trusting that you'll be notified (woken up) the moment the printer frees up, rather than standing there indefinitely accomplishing nothing.

## Why This Hybrid Design Is Necessary

Pure spinning is cheap for very short waits but can be severely wasteful for longer ones (especially on a single-CPU system, where the spinning thread directly steals CPU time from the very thread that would release the lock). Pure sleeping (via a system call, every single time) avoids that waste but pays a real, fixed overhead cost (the system call itself, plus the scheduler's work to block and later wake the thread) even for locks that would have become free almost instantly. The two-phase approach gets the best of both: cheap handling of the common case (short waits, resolved by a brief spin) and safe, non-wasteful handling of the uncommon case (longer waits, resolved by sleeping instead of burning CPU indefinitely).

## Advantages of Two-Phase Locks

- **Cheap for the common case** — most critical sections are short, so a brief spin frequently succeeds without ever needing the more expensive sleep/wake mechanism.
- **Avoids severe waste for longer waits** — falling back to sleeping means a thread waiting on a lock that won't be free for a while consumes no CPU time at all, rather than spinning uselessly.

## Disadvantages / Trade-offs

- **More complex to implement correctly** than a pure spin lock — it requires coordinating both the fast, spin-based path and the OS-mediated sleep/wake path without introducing new race conditions in that coordination itself.
- **The exact spin duration before falling back to sleep is a tunable parameter** — too short, and the cheap common case is under-exploited; too long, and the "avoid wasting CPU on a long wait" benefit is delayed unnecessarily.

## Best Practices

- When choosing (or reasoning about) a lock implementation, consider expected critical-section length and CPU core availability — pure spinning is more defensible for very short critical sections on multi-core systems, while pure sleeping is safer as a general-purpose default, especially on systems with limited cores.
- Recognize the futex-style two-phase approach as the practical, real-world answer used by production operating systems — not merely a textbook compromise, but the actual mechanism behind most general-purpose lock implementations today.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Spinning is always wasteful and should never be used." | On a multi-core system, briefly spinning while the lock-holder is actively running on a different core can be cheaper than the overhead of a full sleep/wake cycle — spinning is genuinely reasonable for short, well-bounded waits, not universally wasteful. |
| "Sleeping is free, so a lock implementation should always immediately put a waiting thread to sleep rather than spin at all." | Sleeping requires a system call and OS-level scheduling involvement (Module 02, Topic 2), which has real, non-zero overhead — for very short waits, this overhead can exceed the cost of simply spinning briefly first, which is exactly why two-phase locks spin before falling back to sleeping. |

## Interview Questions

1. **Q: What is a spin lock, and when is spinning a reasonable strategy?**
   A: A lock where a waiting thread repeatedly retries acquiring it in a tight loop rather than yielding the CPU. Spinning is reasonable when the wait is expected to be very short and (especially on a multi-core system) the lock-holder is actively running on a different core and likely to release it soon.

2. **Q: Why is spinning particularly wasteful on a single-CPU system?**
   A: If the lock-holder isn't currently running (because the spinning thread was scheduled instead), the spinning thread's CPU time accomplishes nothing — it's actively preventing the lock-holder from getting the CPU time it needs to finish and release the lock.

3. **Q: What is a two-phase (futex-based) lock, and why do real systems use this approach?**
   A: A lock that spins briefly first (cheaply handling the common case where the lock frees up almost immediately), then puts the waiting thread to sleep (via a system call, moving it to Blocked) if it's still not free — avoiding both the overhead of always sleeping immediately and the waste of spinning indefinitely on a longer wait.

## Summary

- A spin lock has a waiting thread repeatedly retry acquiring the lock in a tight loop, which is cheap for very short waits but can be seriously wasteful — especially on a single-CPU system, where it directly steals CPU time from the lock-holder.
- A two-phase (futex-based) lock spins briefly first, then falls back to sleeping (an OS-mediated Blocked state) if the lock is still unavailable, combining cheap common-case handling with safe handling of longer waits.
- This is the practical approach real operating systems use for general-purpose locking, not merely a theoretical compromise.
- This closes out the module's coverage of locks — the module summary ties critical sections, hardware-based atomicity, and the spin/sleep trade-off together before Module 14 introduces condition variables and semaphores, tools for coordination beyond simple mutual exclusion.
