# Module 13 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Basic Idea of Locks** — critical sections, mutual exclusion, and the lock/unlock interface, applied directly to Module 01, Topic 4's lost-update bug
- [x] **Building Locks: Hardware Support** — why ordinary instructions can't build a correct lock, and test-and-set/compare-and-swap as the atomic hardware primitives that can
- [x] **Spin Locks vs. Two-Phase Locks** — the real cost of spinning, and the futex-based hybrid approach real systems use

## The Big Picture

This module delivered the first concrete tool for the danger Module 12 set up: threads sharing data with no guaranteed ordering. It did so in three layers — the goal (mutual exclusion over a critical section), the mechanism that makes the goal achievable at all (hardware atomicity), and the practical engineering question of what to do while waiting (spin, sleep, or both).

```
   Module 01, Topic 4: the lost-update problem
   Module 12: threads share data, ordering isn't guaranteed
                          │
                          ▼
   Topic 1: locks — enforce mutual exclusion over a critical section
                          │
                          ▼
   Topic 2: WHY a naive lock is broken, and the hardware fix
            (test-and-set / compare-and-swap)
                          │
                          ▼
   Topic 3: WHAT to do while waiting — spin (cheap, short waits),
            sleep (safe, long waits), or both (real-world futexes)
```

## Practical Connections

- **Why every mainstream programming language's standard library ships a `Mutex`/`Lock` type, and why you should almost always reach for it instead of hand-rolling your own with a plain boolean flag** — Topic 2 is the precise, technical reason: a hand-rolled check-then-set lock is subtly broken, while the standard library's implementation is built on genuine hardware atomicity.
- **Why heavily-contended locks show up as a real, measurable performance bottleneck in profiling tools for multithreaded software** — this is Topic 1's serialization cost and Topic 3's spin/sleep overhead, made concrete and visible.
- **Why "lock-free" and "wait-free" data structures are an active, ongoing area of systems engineering** — they're built directly on compare-and-swap (Topic 2), attempting to avoid Topic 1's serialization cost entirely for specific data structures.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Critical section vs. lock | A critical section is the code that needs protection; a lock is the mechanism used to protect it (enforcing mutual exclusion). |
| Test-and-set vs. compare-and-swap | Test-and-set unconditionally writes a new value and returns the old one. Compare-and-swap only writes the new value if the current value matches an expected one, reporting success or failure — a strictly more general primitive. |
| Spin lock vs. two-phase lock | A spin lock always retries in a tight loop, however long the wait. A two-phase lock spins briefly, then falls back to sleeping (an OS-mediated Blocked state) if the wait continues — combining both strategies. |

## What's Next

Locks solve mutual exclusion — "only one thread here at a time" — but not every concurrency problem is about exclusion alone. Sometimes a thread needs to wait for a specific *condition* to become true (e.g., "wait until there's an item in a queue"), not just wait for a lock to become free. **Module 14 — Condition Variables and Semaphores** introduces exactly these tools, using the classic producer-consumer problem as the motivating example, and semaphores as a single, unifying primitive that can express both mutual exclusion and condition-based waiting.
