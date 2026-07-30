# Module 15 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Non-Deadlock Bugs** — atomicity violations and order violations, and why they're at least as common as deadlock in practice
- [x] **Deadlock: Conditions and Prevention** — the four necessary conditions, mapped onto Module 14's dining philosophers example, and prevention strategies attacking each one
- [x] **Deadlock Avoidance and Detection** — safe states and runtime avoidance, versus letting deadlock happen and recovering from it

## The Big Picture

This module completed the concurrency-bugs picture Module 14, Topic 3 opened concretely. It first corrected a common bias (deadlock isn't the only, or even the most common, concurrency bug), then formalized deadlock precisely, then covered the full spectrum of strategic responses to it.

```
   Module 14, Topic 3: a concrete deadlock, fixed by one specific trick
                          │
                          ▼
   Topic 1: NON-DEADLOCK bugs — atomicity & order violations
            (no cycle, no hang — just silent wrong results)
                          │
                          ▼
   Topic 2: deadlock FORMALIZED — 4 necessary conditions
            PREVENTION: deny one condition (usually: consistent ordering)
                          │
                          ▼
   Topic 3: AVOIDANCE (runtime safe-state checks) vs.
            DETECTION + RECOVERY (let it happen, then fix it)
```

## Practical Connections

- **Why a "flaky test" that fails once in a hundred CI runs is often an atomicity or order violation, not a deadlock** — Topic 1's key point made concrete: these bugs don't hang, they intermittently produce a wrong result, which is exactly what a flaky (not permanently frozen) test looks like.
- **Why style guides for concurrent codebases often include an explicit, mandatory "lock acquisition order" document** — this is Topic 2's most practical prevention strategy, made into an enforced team convention.
- **Why database systems can detect and automatically roll back a "deadlocked transaction" rather than hanging forever** — this is Topic 3's detection-and-recovery strategy, applied at the transaction level instead of the thread level.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Atomicity violation vs. order violation | An atomicity violation breaks an intended indivisible sequence via a missing/inconsistent lock. An order violation assumes an execution order between threads that isn't actually enforced by any synchronization mechanism. |
| Deadlock prevention vs. avoidance | Prevention structurally denies one of the four necessary conditions, making deadlock impossible regardless of runtime behavior. Avoidance allows all four to exist but uses runtime checks against declared future needs to steer around unsafe states. |
| Deadlock avoidance vs. detection | Avoidance proactively refuses any request that could lead to an unsafe state. Detection lets deadlock actually happen, then periodically checks for it and recovers (typically by terminating or rolling back a thread). |

## What's Next

Modules 12–15 covered thread-based concurrency in full: the thread abstraction, locks, condition variables/semaphores, and the bugs that arise from using them incorrectly. **Module 16 — Event-Based Concurrency** covers a genuinely different concurrency model — handling many concurrent tasks using a single thread and an event loop (via select/poll/epoll), rather than one OS thread per concurrent task — and the specific problem (blocking calls) that this model has to solve to work at all.
