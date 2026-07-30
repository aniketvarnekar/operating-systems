# Module 04 — CPU Scheduling Basics

## Module Goal

By the end of this module, you will understand **how the OS decides which ready process actually gets to run next, and for how long** — the policy question Module 03 deliberately left open once it established that the OS can always regain the CPU. You'll learn the metrics used to judge a scheduling policy, and walk through the classic algorithms in the order they were historically developed, each one fixing a specific flaw in the one before it.

## Topics Covered in This Module

1. **[Scheduling Metrics](01-scheduling-metrics.md)** — Turnaround time, response time, and fairness: the yardsticks every scheduling policy is judged against, and why they often conflict.
2. **[First-In, First-Out (FIFO) Scheduling](02-first-in-first-out-scheduling.md)** — The simplest possible policy, and the "convoy effect" flaw it exposes.
3. **[Shortest Job First (SJF)](03-shortest-job-first.md)** — Fixing the convoy effect by running the shortest job first — and the new flaw this creates when jobs arrive at different times.
4. **[Shortest Time-to-Completion First (STCF)](04-shortest-time-to-completion-first.md)** — Making SJF preemptive, and why this is provably optimal for turnaround time — at a real cost to response time.
5. **[Round Robin (RR)](05-round-robin.md)** — Optimizing for response time instead, via short time slices — and the turnaround-time cost this trades away.
6. **[Incorporating I/O](06-incorporating-io.md)** — What changes once jobs aren't purely CPU-bound, and how overlapping I/O with computation keeps the CPU busy.
7. **[Module Summary](07-module-summary.md)** — Consolidated recap, including a comparison table across every policy covered.

## Prerequisites

- Module 03 in full — this module assumes you understand that the OS can always regain the CPU (via traps and timer interrupts) and perform a context switch; this module is entirely about the *policy* decision made once that happens.

## How to Study This Module

Read in order — this module is a single connected narrative, not a list of independent algorithms. Topic 1 gives you the vocabulary to judge everything that follows. Topics 2–5 then walk through FIFO → SJF → STCF → Round Robin as a chain of "here's a real flaw in the previous policy, here's the next policy designed specifically to fix it, here's the new trade-off that fix introduces" — by the end, you'll see that there is no single "best" scheduler, only better or worse fits for a specific goal (turnaround vs. response time vs. simplicity). Topic 6 then relaxes the simplifying assumption (used throughout Topics 2–5) that jobs never perform I/O, showing what a scheduler actually has to account for in the real world.
