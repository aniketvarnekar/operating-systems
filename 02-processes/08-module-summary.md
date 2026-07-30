# Module 02 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Process Abstraction** — program vs. process, the machine state a process encompasses (memory, registers, program counter, open files)
- [x] **Process States and Lifecycle** — Running/Ready/Blocked, and exactly which events trigger each transition
- [x] **The Process Control Block** — what it stores, and how it enables the context switch
- [x] **The fork() System Call** — the "returns twice" behavior, what gets copied, and how parent/child are distinguished
- [x] **The exec() System Call** — replacing a process's running program in place, why it "never returns" on success, what survives it
- [x] **wait() and Process Termination** — collecting a child's exit status, zombies, orphans and reparenting
- [x] **Why fork() and exec() Are Separate** — the I/O-redirection use case that this two-call design elegantly enables

## The Big Picture

This module turned Module 01's abstract promise ("the OS virtualizes the CPU") into something concrete: the **process** is the unit being virtualized, the **PCB** is how the OS remembers each one, and **states** (Running/Ready/Blocked) describe what a process is doing at any instant. The **fork/exec/wait** trio is the actual API real programs (most visibly, your shell) use to create, transform, and clean up these units.

```
        THE PROCESS LIFECYCLE, END TO END
        ──────────────────────────────────

  fork()  ──►  child is a copy of parent  ──►  exec()  ──►  child now runs
  (Topic 4)     (independent address space,     (Topic 5)    a different
                 own PCB, own PID)                            program entirely
                                                                    │
                                                                    ▼
                                                          process runs, moving
                                                          between Running/Ready/
                                                          Blocked (Topic 2)
                                                                    │
                                                                    ▼
                                                          process terminates
                                                          → becomes a zombie
                                                          until parent calls
                                                          wait()  (Topic 6)
```

## Practical Connections

- **Every command you type into a shell** goes through exactly this fork → (redirect setup) → exec → wait cycle — you've now seen the actual mechanism behind something you use every day.
- **`top`/`ps`/Activity Monitor/Task Manager** are all just tools that read the OS's collection of PCBs (Topic 3) and display each process's state, PID, and resource usage in human-readable form.
- **A server process that "leaks zombies"** under load (visible as a slowly growing process count) is a direct, real-world instance of a parent forking many children (e.g., one per connection) without reliably calling `wait()` on each one (Topic 6).
- **Container tools (Docker, etc.)** rely on exactly this same process model underneath — a container is, at its core, one or more regular OS processes given extra isolation, not a fundamentally different kind of execution unit.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Program vs. process | A program is a static file on disk; a process is a specific, live, running instance of it, with its own memory and register state. |
| Ready vs. Blocked | Ready means "could run right now, just waiting for a scheduling turn"; Blocked means "can't usefully run yet regardless of CPU, waiting on an external event like I/O." |
| fork() vs. exec() | fork() creates a new, independent process (a copy of the caller); exec() does not create any process — it replaces the calling process's own running program in place, keeping the same PID. |
| Zombie vs. orphan | A zombie is a *terminated* process awaiting its parent's `wait()` call. An orphan is a *still-running* child whose parent terminated first, and which gets reparented to a system reaper process. |

## What's Next

Module 02 defined the process abstraction and its lifecycle API. It deliberately left one question open: when the OS switches the CPU from one process to another (a context switch), how does that actually happen at the hardware/software boundary, and how does the OS guarantee it retains control of the CPU at all, rather than a process simply refusing to give it back? **Module 03 — Direct Execution** answers exactly that: the low-level mechanism (limited direct execution, traps, timer interrupts) that makes running many processes on one physical CPU both fast and safe.
