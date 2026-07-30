# Module 02 — Processes

## Module Goal

By the end of this module, you will understand **what a process actually is, the states it moves through during its life, the kernel data structure that tracks it, and the exact system-call API real programs use to create and manage processes**. This module is the first deep dive into CPU virtualization (introduced conceptually in Module 01, Topic 3) — it defines the abstraction; Module 03 explains the low-level mechanism that makes running many processes on one CPU physically possible, and Modules 04–05 cover the scheduling policies that decide which process runs when.

## Topics Covered in This Module

1. **[The Process Abstraction](01-the-process-abstraction.md)** — What a process is, precisely, and why it's more than just "a running program."
2. **[Process States and Lifecycle](02-process-states-and-lifecycle.md)** — The states a process moves through (running, ready, blocked) and the events that trigger each transition.
3. **[The Process Control Block](03-the-process-control-block.md)** — The kernel data structure that holds everything the OS needs to know about a process, and how it enables switching between processes at all.
4. **[The fork() System Call](04-the-fork-system-call.md)** — How new processes are actually created on UNIX-like systems, and its famously surprising "returns twice" behavior.
5. **[The exec() System Call](05-the-exec-system-call.md)** — How a process replaces its own running program with a different one entirely.
6. **[wait() and Process Termination](06-wait-and-process-termination.md)** — How a parent process waits for a child to finish, reaps its exit status, and what a "zombie" process actually is.
7. **[Why fork() and exec() Are Separate](07-why-fork-and-exec-are-separate.md)** — The specific, elegant problem this two-call design solves that a single combined "create and run" call could not.
8. **[Module Summary](08-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 01 in full, especially Topic 3 (Virtualization) and Topic 6 (System Calls) — processes are the concrete embodiment of "virtualizing the CPU," built entirely out of system calls.

## How to Study This Module

Read in order. Topics 1–3 build the conceptual and data-structure foundation — what a process *is* and how the OS tracks it. Topics 4–6 are hands-on: the actual system calls (`fork`, `exec`, `wait`) that every UNIX-like OS (Linux, macOS) exposes for process management, including `fork`'s infamous "returns twice" surprise that trips up nearly everyone the first time they see it. Topic 7 is the payoff — it explains *why* UNIX chose to split process creation into two separate calls instead of one, a design decision that looks strange at first and turns out to be one of the most elegant ideas in OS history once you see the problem it solves (redirecting a shell command's output to a file, without modifying the target program at all).
