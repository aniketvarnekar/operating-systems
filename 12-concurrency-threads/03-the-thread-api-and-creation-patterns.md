# The Thread API and Creation Patterns

## Learning Objectives

By the end of this section you should be able to:
- Describe what thread creation and thread joining do, in terms of the underlying API
- Trace through a simple multithreaded program and predict its possible outcomes
- Explain why the order of execution between independently-created threads is not guaranteed

## Prerequisites

- Topic 2 (The Thread Abstraction)
- Module 02, Topic 6 (wait() and Process Termination) — a useful point of comparison

## Motivation

Topics 1–2 built the conceptual model; this topic makes it concrete with the actual function calls real multithreaded programs use, and — just as importantly — trains you to predict what such a program *can't* guarantee about execution order, which is the seed of every concurrency bug covered starting in Module 15.

## Problem Statement

Given the thread abstraction from Topic 2, how does a program actually bring a new thread into existence, running some specific function concurrently with the thread that created it? And once created, how does the creating thread find out when a spawned thread has finished, similar to how a parent process uses `wait()` to learn when a child process terminates (Module 02, Topic 6)?

## Concept

### Thread Creation

> Creating a thread (commonly via a call like `pthread_create()` in the POSIX threads API) takes a function to run and starts a **new, independent thread of execution within the same process**, beginning execution at that function — while the calling thread continues running its own code immediately afterward, without waiting for the new thread to do anything first.

This is meaningfully different from `fork()` (Module 02, Topic 4): `fork()` creates an entirely new process with its own copy of the address space; thread creation adds a new independent sequence of execution *within* the same process's existing address space (Topic 2), with no address-space copying involved at all.

```c
pthread_create(&thread_id, NULL, my_function, my_argument);
// the calling thread continues running immediately — it does
// NOT wait for my_function to start or finish
```

### Thread Joining

> **Joining** a thread (via `pthread_join()`) blocks the calling thread until the specified thread has finished executing, optionally retrieving a return value from it — directly analogous to how `wait()` lets a parent process block until a specific child process terminates (Module 02, Topic 6).

```c
pthread_join(thread_id, &return_value);
// blocks HERE until the specified thread finishes
```

Without an explicit join (or an equivalent synchronization mechanism, Modules 13–14), there is no guarantee about the relative timing between a creating thread's continued execution and a newly-created thread's progress — exactly the same kind of nondeterminism Module 02, Topic 4 flagged for `fork()`'s parent/child ordering, now applying to threads instead.

### Worked Example: Unpredictable Interleaving

Consider a program that creates two threads, each simply printing a message:

```c
void *print_A(void *arg) { printf("A"); return NULL; }
void *print_B(void *arg) { printf("B"); return NULL; }

pthread_create(&t1, NULL, print_A, NULL);
pthread_create(&t2, NULL, print_B, NULL);
pthread_join(t1, NULL);
pthread_join(t2, NULL);
```

This program might print `AB`, or it might print `BA` — **both are entirely valid, correct outcomes**. Nothing in this code specifies which thread's `printf` call actually executes first; that's determined by the OS scheduler (Modules 04–05) at runtime, and can vary from one run of the exact same program to the next. This is a direct, concrete instance of Module 01, Topic 4's warning: concurrent code's correctness cannot depend on assuming one particular interleaving unless that ordering is explicitly enforced (Modules 13–14).

### Common Thread Creation Patterns

- **Fork-join pattern**: create several threads to work on independent pieces of a larger problem, then join all of them before combining their results — directly exploiting Topic 1's Motivation 1 (using multiple CPU cores for one logical program).
- **Worker/thread-pool pattern**: create a fixed set of threads up front that repeatedly pull tasks from a shared queue, rather than creating and destroying a new thread for every single small unit of work — avoiding the real (though smaller than a process's) overhead of repeated thread creation and teardown.

## Internal Working (Preview)

```
   Main thread:
        │
        ├── pthread_create(print_A) ──► [ Thread 1 starts running print_A ]
        │                                       (independently, concurrently)
        ├── pthread_create(print_B) ──► [ Thread 2 starts running print_B ]
        │                                       (independently, concurrently)
        │
        ├── pthread_join(t1) ──────► blocks until Thread 1 finishes
        ├── pthread_join(t2) ──────► blocks until Thread 2 finishes
        ▼
   Main thread continues, now that both have finished

   Relative order of "A" and "B" printing: NOT GUARANTEED —
   depends on scheduler decisions (Modules 04–05) at runtime
```

## Real-World Analogy

Think of thread creation like a manager (the main thread) handing off two separate tasks to two different employees (new threads) and then immediately going back to their own work, without waiting to see who starts first — both employees begin working independently, at whatever pace and order the office's own scheduling happens to allow, entirely outside the manager's control. Joining is the manager later deliberately pausing their own work specifically to wait until a particular employee reports back that their task is fully done, before proceeding — exactly as a parent process's `wait()` (Module 02, Topic 6) blocks for a specific child.

## Why Execution Order Is Not Guaranteed

Just as Module 02, Topic 4 established that a fork()'d parent and child have no guaranteed relative execution order without explicit synchronization, newly-created threads have exactly the same property: the OS scheduler (Modules 04–05) decides, at runtime, which ready thread actually runs at any given moment, and this decision can vary between runs of the identical program, depending on scheduling policy details, system load, and other factors entirely outside the program's own control. Treating any particular interleaving as guaranteed, without explicit synchronization, is a direct, common source of the concurrency bugs Module 15 catalogs.

## Advantages of This API Design

- **Simple, explicit creation and synchronization primitives** — create and join map directly onto "start something independent" and "wait for something specific to finish," mirroring the already-familiar fork/wait pattern from Module 02.
- **Doesn't impose a specific ordering unnecessarily** — letting the scheduler freely interleave independently-created threads is what allows the performance benefits from Topic 1 (using multiple cores, overlapping I/O waits) to actually materialize.

## Disadvantages / Risks

- **The very flexibility that enables performance benefits also enables race conditions** — since ordering isn't guaranteed by default, any code that implicitly assumes a specific interleaving without explicit synchronization is fragile and can fail unpredictably, exactly matching Module 01, Topic 4's description of race conditions.
- **Forgetting to join a thread** can create issues analogous to Module 02, Topic 6's zombie processes — resources associated with a finished-but-unjoined thread may not be fully cleaned up, depending on the specific threading implementation's semantics.

## Best Practices

- Never write multithreaded code that implicitly assumes a specific execution order between independently-created threads unless that order is explicitly enforced via a synchronization mechanism (Modules 13–14) — assume any valid interleaving is possible, and design accordingly.
- Reliably join every thread you create (or otherwise account for its lifecycle) — just as reliably calling `wait()` for every forked child (Module 02, Topic 6) avoids zombie accumulation, doing the equivalent for threads avoids analogous resource-cleanup issues.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Threads created one after another in source code will execute in that same order." | The scheduler decides actual execution order at runtime, independent of the order threads were created in source code — both orderings (and interleavings) are valid outcomes unless explicit synchronization enforces a specific order. |
| "pthread_create() waits for the new thread to actually start running before returning." | It returns immediately, with the calling thread continuing its own execution — the newly created thread begins running independently, at a time entirely determined by the scheduler, not necessarily immediately. |

## Interview Questions

1. **Q: What does creating a thread do, and how is it different from fork()?**
   A: Thread creation starts a new, independent sequence of execution within the *same* process's existing address space, with the calling thread continuing immediately afterward. fork() instead creates an entirely new, separate process with its own copy of the address space (Module 02, Topic 4) — thread creation involves no address-space copying at all.

2. **Q: What does joining a thread do, and what is it analogous to from Module 02?**
   A: Joining blocks the calling thread until the specified thread finishes, optionally retrieving its return value — directly analogous to how wait() lets a parent process block until a specific child process terminates.

3. **Q: In a program that creates two threads to each print a different message, is the relative order of their output guaranteed?**
   A: No — the OS scheduler determines actual execution order at runtime, and either thread's output could appear first; both orderings are valid outcomes unless the program explicitly enforces a specific order via synchronization (Modules 13–14).

## Summary

- Thread creation starts a new, independent execution sequence within the same process's address space, with the calling thread continuing immediately; joining blocks until a specific thread finishes, mirroring fork()/wait() from Module 02.
- The relative execution order between independently-created threads is never guaranteed by default — it's determined by the scheduler at runtime and can vary between runs of the same program.
- Common patterns (fork-join, worker/thread-pool) structure multithreaded programs around this creation/join API to exploit multiple cores or overlap I/O waits.
- This closes out the module's introduction to threads — the module summary ties motivation, the sharing model, and the API together before Module 13 introduces locks, the first concrete tool for taming the race conditions this topic's unpredictable interleaving makes possible.
