# The Event Loop and select()/poll()

## Learning Objectives

By the end of this section you should be able to:
- Explain the motivation for event-based concurrency as an alternative to one-thread-per-task
- Explain what select()/poll() do, at a conceptual level
- Explain why event-based concurrency avoids most of Module 15's shared-state concurrency bugs

## Prerequisites

- Module 12, Topic 1 (Why Threads?)
- Module 04, Topic 6 (Incorporating I/O)

## Motivation

Modules 12–15 built an entire toolkit around one specific concurrency model: multiple OS threads sharing an address space, coordinated with locks and condition variables. This topic introduces a genuinely different model — one thread, handling many concurrent tasks by itself — and the specific motivation for choosing it.

## Problem Statement

Recall Module 14, Topic 1 (the C10K-style motivation, previewed there): a server handling many thousands of concurrent client connections, most of which are idle most of the time (waiting on the next request, or a slow network response), needs *some* way to make progress on whichever connections currently have data ready, without wasting resources on the ones that don't. One-thread-per-connection (Module 12) works, but creating and context-switching between thousands of OS threads has real, non-trivial overhead (Module 03, Topic 4; Module 12, Topic 1) — and every one of those threads sharing data needs the entire Module 13–15 toolkit (locks, condition variables, deadlock awareness) just to stay correct. Is there a way to handle many concurrent tasks with dramatically less overhead, and without needing locks at all?

## Concept

### The Event Loop

> An **event loop** is a single-threaded execution model that repeatedly: (1) checks which of several registered event sources (e.g., open network sockets) currently have something ready to be handled, (2) handles each ready event's work (typically quickly), and (3) repeats — rather than dedicating a separate thread to each source and letting the OS scheduler (Modules 04–05) interleave them.

Because there is only **one** thread actually executing application code at any moment (even if it's managing many logical tasks), there is no possibility of two pieces of application-level code running *simultaneously* on the same shared data — which is precisely what eliminates the vast majority of Module 15's shared-state concurrency bugs (atomicity violations, and most deadlock scenarios) for this specific model, without needing a single lock.

### select() and poll(): Checking Readiness

> **select()** and **poll()** are system calls that let a single thread ask the OS, in one call, "of this entire list of file descriptors (open sockets, files, etc.) I'm monitoring, which ones currently have data ready to read (or are ready to write, or have some other event pending)?" — blocking only until at least one is ready, rather than blocking on any single one individually.

This is the mechanism that makes the event loop viable at all: instead of a thread blocking on one specific socket's read call (which would stall the entire single-threaded program if that particular socket has nothing to do while others are ready), the thread blocks on `select()`/`poll()` itself, which returns as soon as **any** monitored source has something ready — the loop then handles just that ready work and immediately checks again.

```c
while (1) {
    ready_set = select(all_monitored_sockets);
    for (socket in ready_set) {
        handle_ready_event(socket);   // quick, non-blocking work
    }
}
```

### Why This Avoids Most Shared-State Concurrency Bugs

Because only one thread ever executes application logic at a time in this model, there is no genuine simultaneous access to shared, in-memory application data from two pieces of application code at once — the entire premise of Module 15's atomicity violations (an unprotected access racing against a locked one) simply doesn't apply, since there's no second thread to race against. This is a real, substantial advantage: a large category of extremely hard-to-debug bugs (Module 01, Topic 4; Module 15, Topic 1) is sidestepped by construction, not merely mitigated by careful locking discipline.

## Internal Working (Preview)

```
   Thread-per-connection model (Module 12):        Event loop model (this module):

   Thread 1 ── blocks on Connection 1's read()      ONE thread:
   Thread 2 ── blocks on Connection 2's read()        loop {
   Thread 3 ── blocks on Connection 3's read()          ready = select(conn1, conn2, conn3, ...)
   ...                                                    for each ready connection:
   (OS scheduler switches among many threads,               handle it (quickly)
    each with its own stack + TCB, Module 12,             }
    Topic 2 — real overhead per thread)

                                                     (ONE thread, ONE stack — select()
                                                      blocks only until SOMETHING
                                                      is ready, then loop handles it)
```

## Real-World Analogy

Think of a single receptionist (one thread) managing many ongoing phone calls on hold (many connections), versus hiring one dedicated employee per caller (one thread per connection). The receptionist doesn't dedicate their full, undivided attention to any one caller while the others wait indefinitely — instead, they periodically check a single switchboard display (select()/poll()) that lights up exactly which calls currently need attention right now, quickly handles each one that's actually ready, and checks the display again. This one receptionist can handle far more simultaneous calls than would be practical to hire a dedicated employee for each — as long as no single call ever demands their continuous, blocking attention for a long stretch (the exact problem Topic 2 addresses).

## Why This Design Is Useful

For workloads dominated by many mostly-idle connections waiting on I/O (Module 04, Topic 6's I/O-bound bursts), one-thread-per-task wastes substantial resources on threads that spend nearly all their time Blocked (Module 02, Topic 2), and forces every shared-data access through Module 13–15's locking discipline. An event loop instead uses exactly one thread's worth of resources, checking many sources' readiness cheaply via select()/poll(), and needs no locking at all for its own single-threaded application logic — a fundamentally different, often more resource-efficient answer to the same "handle many concurrent things" goal Module 12 introduced.

## Advantages of the Event Loop Model

- **Dramatically lower per-task overhead** than one-thread-per-task, since there's no separate stack, TCB (Module 12, Topic 2), or context-switching cost per logical task.
- **Eliminates most shared-state concurrency bugs by construction** — since only one thread ever executes application code, there's no simultaneous access to shared data to race on.

## Disadvantages / Limitations (Previewed)

- **Cannot exploit multiple CPU cores for a single event loop** — since there's fundamentally only one thread of application-level execution, this model alone doesn't gain Module 12, Topic 1's "use multiple cores" benefit the way a multithreaded design can (real systems often combine multiple event-loop processes/threads to address this).
- **Every single event handler must be quick and non-blocking** — a single slow or blocking operation anywhere freezes the *entire* loop, since there's no other thread to pick up the slack. This specific, serious limitation is the subject of the next topic.

## Best Practices

- Reach for an event-loop model specifically for I/O-bound workloads with many mostly-idle concurrent tasks (network servers being the classic case); reach for thread-based concurrency (Module 12) when tasks are CPU-bound and need to exploit multiple cores, or when blocking, long-running operations are unavoidable within individual tasks.
- Never perform a long-running, blocking operation directly inside an event handler — doing so stalls every other task the loop is managing, since there's no other thread available to continue their work.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Event-based concurrency eliminates concurrency bugs entirely." | It eliminates most *shared in-memory data* races specific to the single-threaded application logic, since only one thread executes it at a time — but it doesn't eliminate concurrency bugs at the system level (e.g., ordering issues across multiple event-loop processes) or make blocking calls harmless (Topic 2). |
| "select()/poll() run each monitored event handler concurrently." | select()/poll() only report which sources are ready; the single event-loop thread then handles each ready event one at a time, sequentially — there's no concurrent execution of handlers happening at all. |

## Interview Questions

1. **Q: What is an event loop, and how does it differ from the thread-per-task model from Module 12?**
   A: An event loop uses a single thread that repeatedly checks which of many registered event sources currently have work ready, and handles each ready source's work in turn — rather than dedicating a separate OS thread (with its own stack and context-switching overhead) to each concurrent task.

2. **Q: What do select() and poll() do?**
   A: They let a single thread ask the OS, in one call, which of a list of monitored file descriptors currently have data ready (or are ready for some other event), blocking only until at least one is ready — rather than blocking on any single source individually.

3. **Q: Why does the event loop model avoid most of Module 15's shared-state concurrency bugs?**
   A: Because only one thread ever executes application-level code at a time, there's no possibility of two pieces of application code accessing shared in-memory data simultaneously — the entire premise behind atomicity violations and most deadlock scenarios simply doesn't apply.

## Summary

- An event loop handles many concurrent tasks with a single thread, using select()/poll() to check which registered sources currently have work ready, rather than dedicating an OS thread per task.
- This dramatically reduces per-task overhead and eliminates most shared-state concurrency bugs by construction, since only one thread ever executes application logic.
- The model's core requirement — every event handler must be quick and non-blocking — is also its sharpest limitation, covered in the next topic alongside epoll, the scalable evolution of select/poll.
