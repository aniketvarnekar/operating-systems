# Module 12 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Why Threads?** — exploiting multiple CPU cores and overlapping I/O waits within a single process, and why threads are cheaper than processes
- [x] **The Thread Abstraction** — what's shared (code, heap, globals) versus private (registers, stack) per thread, and the Thread Control Block
- [x] **The Thread API and Creation Patterns** — pthread_create/pthread_join, and why execution order between threads is never guaranteed by default

## The Big Picture

This module introduced Concurrency's foundational unit — the thread — by directly contrasting it with the process abstraction Modules 02–05 already established. The core tension this module sets up, and that the rest of Concurrency (Modules 13–16) exists to resolve, is this: threads are useful *because* they share an address space, but that same sharing is exactly what makes uncoordinated concurrent access dangerous.

```
   Module 02: Process               Module 12: Thread
   ─────────────────                ─────────────────
   own private address space   vs.  SHARES process's address space
   own PCB                          own TCB (registers, stack) +
                                     shared PCB (address space, files)
   fork()/exec()/wait()             pthread_create()/pthread_join()

   Isolation is the whole point            Sharing is the whole point
   (Module 06, Topic 1)                    (this module, Topic 1)
                                                    │
                                                    ▼
                              Shared data + no guaranteed ordering
                              (Topic 3) = the exact setup for race
                              conditions (Module 01, Topic 4)
                                                    │
                                                    ▼
                              Modules 13–16: the tools that make
                              shared-data access SAFE
```

## Practical Connections

- **Why a web server or database can serve many requests "simultaneously" using only a modest number of CPU cores, while still sharing one in-memory cache across all of them** — this is Topic 1's exact motivation (shared data, multiple cores/overlapped I/O) made concrete.
- **Why a multithreaded program can produce different output (or fail intermittently) across separate runs on the exact same input** — this is Topic 3's "execution order is not guaranteed" finding, directly experienced rather than just described.
- **Why thread pools (reusing a fixed set of worker threads) are so common in real server software, rather than spawning a brand-new thread per request** — Topic 3's worker/thread-pool pattern exists specifically to amortize thread-creation overhead across many short-lived tasks.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Process vs. thread | A process has its own entire, private address space. A thread runs within an existing process, sharing that process's address space with any other threads in it. |
| PCB vs. TCB | A PCB holds process-wide information (address space, open files), shared by all threads in that process. A TCB holds one specific thread's private execution state (registers, stack pointer) — a process has one PCB but potentially many TCBs. |
| pthread_create() vs. fork() | Thread creation starts a new execution sequence within the same address space, with no copying involved. fork() creates an entirely new process with its own copied address space. |

## What's Next

This module established that threads share data directly, and that their execution order isn't guaranteed — precisely the ingredients for the race conditions Module 01, Topic 4 first warned about. **Module 13 — Locks** introduces the first, most fundamental tool for taming this: a mechanism that lets a thread declare "no one else may touch this shared data until I say I'm done," built from specific hardware instructions (test-and-set, compare-and-swap) up through spinlocks, ticket locks, and futex-based locks.
