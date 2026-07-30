# wait() and Process Termination

## Learning Objectives

By the end of this section you should be able to:
- Explain what `wait()` does and why a parent process typically calls it
- Explain precisely what a "zombie" process is and why it exists
- Explain what an "orphan" process is and how the OS handles it

## Prerequisites

- Topic 4 (The fork() System Call)

## Motivation

Creating a child process (Topic 4) is only half the story — a well-behaved parent process usually needs to know *when* and *how* that child finished, so it can react accordingly (a shell needs to know when a command finishes so it can print the next prompt; a build tool needs to know if a compilation step succeeded or failed). `wait()` is the mechanism for exactly this, and it comes with two famous, related edge cases — zombies and orphans — that are common interview topics precisely because they reveal how carefully process termination has to be handled.

## Problem Statement

When a child process finishes executing (via a normal return from its `main` function, or by explicitly calling `exit()`), it doesn't just vanish without a trace. Two questions need answering:

1. How does the **parent** find out that the child is done, and what its exit status was (did it succeed, or fail, and with what specific code)?
2. What happens to the child's own bookkeeping (its PCB, Topic 3) in the meantime — does it get cleaned up immediately, or does something need to happen first?

## Concept

### The wait() System Call

> `wait()` (and its more flexible relative, `waitpid()`) is a system call a parent process uses to **block** until one of its child processes terminates, at which point it returns that child's Process ID and (via an output parameter) the child's exit status — the value the child returned from `main()` or passed to `exit()`.

If the child has *already* terminated by the time the parent calls `wait()`, it returns immediately with that child's information — there's no need for parent and child to have any special extra timing coordination beyond this one call.

### The Zombie State

Here's the subtlety that makes this topic interview-famous: when a child process terminates, the OS does **not** immediately erase all trace of it. Instead, the child enters a special terminated-but-not-yet-cleaned-up state, commonly called a **zombie**, in which:

- The child's actual work is completely done — it no longer executes any code, and its memory (address space) has been released.
- Its PCB (Topic 3), specifically the part recording its exit status, is deliberately kept around.

> A **zombie process** is a terminated process whose exit status has not yet been collected by its parent via `wait()`.

The reason for this deliberate half-cleanup is direct: the parent might genuinely need that exit status later (it doesn't know in advance exactly when it will get around to calling `wait()`), so the OS cannot discard it the instant the child finishes — it must be preserved until the parent explicitly retrieves it. Once the parent calls `wait()` and collects the exit status, the zombie is fully removed — this final cleanup step is often called **reaping**.

### Orphan Processes

The reverse problem: what if the **parent** terminates *before* its child does? The child is now an **orphan** — its original parent no longer exists to eventually call `wait()` on it. Rather than leaving orphans permanently unreapable, UNIX-like systems automatically **reparent** an orphan to a special, always-running system process (traditionally process ID 1, `init`, or a similar designated reaper process), which periodically calls `wait()` on all of its (possibly inherited) children, ensuring every process eventually gets properly reaped no matter what happens to its original parent.

## Internal Working (Preview)

```
 Child finishes running
        │
        ▼
 Child becomes a ZOMBIE
 (work is done; exit status
  preserved in its PCB)
        │
        │   parent calls wait()
        ▼
 Parent receives child's PID
 + exit status
        │
        ▼
 Zombie's PCB is fully
 removed ("reaped")


 If the PARENT terminates first, while the child is still running:
        │
        ▼
 Child is reparented to
 the system's designated
 reaper process (e.g., init)
        │
        ▼
 That reaper process will
 eventually call wait() on
 the child when it finishes
```

## Real-World Analogy

Think of a zombie process like a completed job application sitting in an "awaiting sign-off" folder: the applicant (the child) has already finished everything they need to do — submitted, done — but the file (the exit-status record) can't be thrown away yet, because their manager (the parent) hasn't yet reviewed and signed off on it. The folder takes up a small amount of shelf space (kernel memory) until that sign-off (a `wait()` call) happens, at which point the file is finally archived away for good (reaped). An orphan is like an applicant whose original manager quit the company before ever reviewing the file — HR (the system's reaper process, like `init`) automatically takes over responsibility for eventually reviewing and closing out that file instead, so it's never permanently stuck in limbo.

## Why This Design Is Necessary

The OS cannot know, at the exact moment a child terminates, whether or when its parent will get around to asking for its exit status — the parent might be busy doing other work first. Immediately discarding the exit status the instant the child finishes would make it impossible for a parent to ever reliably learn how its child concluded. Keeping a minimal record (the zombie) until the parent explicitly asks for it (`wait()`) is the only way to guarantee that information isn't lost to a race between "child finishes" and "parent gets around to checking."

## Advantages of This Design

- **Reliable exit-status delivery** — a parent is guaranteed to be able to retrieve its child's exit status no matter how long it takes to call `wait()`, since the OS holds onto it in the meantime.
- **No orphaned bookkeeping left forever** — the reparenting-to-init mechanism guarantees every process is eventually reaped, even if its original parent disappears first.

## Disadvantages / Real Dangers

- **Zombie accumulation is a real, practical bug** — a parent process that creates many children but never calls `wait()` on any of them leaves a growing pile of zombie entries consuming kernel resources (a real, historically common source of resource leaks in long-running server processes).
- **Requires deliberate parent code** — a parent must actually remember to call `wait()`/`waitpid()`; the OS won't do it automatically on the parent's behalf while the parent is still alive.

## Best Practices

- Any long-running program that forks child processes should reliably call `wait()`/`waitpid()` for every child it creates — including handling the case where a child might terminate at an unpredictable time relative to the parent's own work (often done via a signal handler for `SIGCHLD` in real systems).
- When debugging a growing "zombie process" count on a running system, the direct cause is almost always a parent process that isn't calling `wait()` promptly (or at all) on its terminated children.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A zombie process is a bug or a crash." | It's an expected, normal, deliberately-designed state — a terminated child simply waiting for its parent to collect its exit status. It only becomes a practical problem if the parent never calls wait() at all, letting zombies accumulate indefinitely. |
| "If a parent dies before its child, the child becomes unmanageable/lost." | The child is automatically reparented to a designated system reaper process (traditionally init), which will eventually call wait() on it — it's never permanently orphaned in a way that prevents cleanup. |
| "wait() and exec() are related calls that do similar things." | They're unrelated in purpose: exec() replaces a process's running program (Topic 5); wait() lets a parent block until a child terminates and collect its exit status. They're often used near each other in code, but they solve entirely different problems. |

## Interview Questions

1. **Q: What does wait() do?**
   A: It's a system call that blocks the calling (parent) process until one of its child processes terminates, then returns that child's PID and its exit status.

2. **Q: What is a zombie process, exactly?**
   A: A process that has finished executing, but whose exit status the OS is still holding onto because its parent hasn't yet called wait() to collect it. It's a deliberate, temporary state, not a bug by itself.

3. **Q: What happens to a child process if its parent terminates before the child does?**
   A: The child becomes an orphan and is automatically reparented to a designated system process (traditionally init/PID 1), which will eventually call wait() on it, ensuring it's still properly reaped despite its original parent no longer existing.

4. **Q: Why can excessive zombie processes be a real problem on a long-running system?**
   A: Each zombie retains a small amount of kernel bookkeeping (its exit status and PCB remnants) until reaped; a parent that creates many children but never calls wait() lets these accumulate indefinitely, consuming kernel resources over time.

## Summary

- wait() lets a parent block until a child terminates, retrieving that child's PID and exit status.
- A zombie is a terminated process whose exit status hasn't yet been collected by its parent — a normal, deliberate, temporary state, not a bug on its own; it becomes a real problem only if the parent never calls wait().
- An orphan (a child whose parent terminated first) is automatically reparented to a system reaper process, guaranteeing it still eventually gets reaped.
- Together, fork() (Topic 4), exec() (Topic 5), and wait() (this topic) form the complete lifecycle API for creating, transforming, and cleaning up processes — the next topic explains exactly why fork and exec were deliberately kept as two separate calls instead of one.
