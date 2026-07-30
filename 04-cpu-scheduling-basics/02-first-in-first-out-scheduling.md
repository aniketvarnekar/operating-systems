# First-In, First-Out (FIFO) Scheduling

## Learning Objectives

By the end of this section you should be able to:
- Describe how FIFO scheduling works
- Calculate average turnaround time for a simple set of jobs under FIFO
- Explain the "convoy effect" with a concrete example, and why it makes FIFO perform poorly in a common, realistic scenario

## Prerequisites

- Topic 1 (Scheduling Metrics)

## Motivation

FIFO is the simplest possible scheduling policy, and a natural starting point — but its simplicity hides a serious, common-case flaw that directly motivates every other policy in this module. Understanding exactly *why* FIFO fails is more valuable than the algorithm itself.

## Problem Statement

Suppose three jobs — A, B, and C — all arrive at (roughly) the same time, ready to run. The simplest imaginable policy is to just run them in the order they arrived, each one to completion, before starting the next. What could go wrong with something this simple?

## Concept

### The Policy

> **First-In, First-Out (FIFO)** — also called First-Come-First-Served (FCFS) — runs jobs strictly in the order they arrive, running each one to full completion before starting the next.

It's also **non-preemptive**: once a job starts running, it keeps the CPU until it finishes (or blocks on I/O — Topic 6), regardless of what else arrives in the meantime.

### Worked Example

Suppose three jobs, each needing 10 seconds of pure CPU time, all arrive at time 0, in order A, B, C:

```
 Time:     0        10        20        30
           │─────A─────│─────B─────│─────C─────│
```

- A completes at t=10 → turnaround = 10 − 0 = 10
- B completes at t=20 → turnaround = 20 − 0 = 20
- C completes at t=30 → turnaround = 30 − 0 = 30
- **Average turnaround time = (10 + 20 + 30) / 3 = 20**

This looks entirely reasonable so far — every job needed the same amount of time, so this is arguably the fairest possible outcome for this specific case.

### The Convoy Effect

Now change just one detail: suppose A needs **100** seconds of CPU time instead of 10, while B and C still only need 10 seconds each, and all three still arrive at time 0:

```
 Time:     0                    100      110      120
           │────────A────────────│───B───│───C───│
```

- A completes at t=100 → turnaround = 100
- B completes at t=110 → turnaround = 110
- C completes at t=120 → turnaround = 120
- **Average turnaround time = (100 + 110 + 120) / 3 = 110**

Both B and C — short jobs that could have finished almost instantly — are forced to wait behind the one long job, purely because of arrival-order bad luck. This is the **convoy effect**: a small number of short jobs get stuck waiting behind one long job, dragging down average turnaround time dramatically, even though swapping the order (running the short jobs first) would have served everyone better on average.

> The **convoy effect** describes the situation where several short jobs queue up behind one long-running job under a non-preemptive, arrival-order-based policy, causing the short jobs' turnaround times to suffer far more than necessary.

## Internal Working (Preview)

```
 FIFO with A(100), B(10), C(10), all arriving at t=0:

   A runs 0→100  (B and C are Ready, but FIFO enforces arrival order —
                   neither gets to run, no matter how short they are)
   B runs 100→110
   C runs 110→120

 Compare to running short jobs first (previewed — Topic 3, SJF):
   B runs 0→10
   C runs 10→20
   A runs 20→120
   Average turnaround = (10 + 20 + 120) / 3 = 50   ← much better than 110
```

This comparison is the exact motivation for Topic 3 (Shortest Job First).

## Real-World Analogy

Think of a single grocery store checkout lane with strict first-come-first-served ordering, and no express lane. If the first person in line has a completely full cart (a long job), everyone behind them — even someone holding just a single item (a short job) — must wait for that entire full cart to be processed before they can even start, no matter how trivially fast their own checkout would be. The rigid "whoever arrived first goes first" rule, applied with no regard for how long each transaction will take, is exactly what produces the convoy effect.

## Why FIFO's Flaw Matters in Practice

Real workloads very commonly mix job lengths unpredictably — a long-running batch computation might start moments before a user launches a quick, interactive command. Under FIFO, that quick command would be forced to wait behind the entire long job, producing exactly the kind of frustrating, unnecessary delay the convoy effect describes — a serious practical problem, not just a theoretical worst case.

## Advantages of FIFO

- **Extreme simplicity** — trivial to implement and reason about; no information about job length is even needed.
- **No starvation** — every job is guaranteed to eventually run, in a predictable, easily-understood order; no job can be perpetually skipped over.

## Disadvantages of FIFO

- **The convoy effect** — short jobs can be forced to wait an excessive, unnecessary amount of time behind one long job, dragging down average turnaround time.
- **No responsiveness for interactive jobs at all** — being non-preemptive, FIFO offers terrible response time (Topic 1) for any job unlucky enough to arrive while a long job is already running; it must wait for that entire job to fully finish before even starting.

## Best Practices

- Recognize FIFO's convoy-effect weakness as the single most important reason more sophisticated scheduling policies exist — it's the direct motivating example for Shortest Job First (Topic 3).
- FIFO can still be a reasonable choice specifically when job lengths are known to be roughly uniform, or when absolute implementation simplicity outweighs performance concerns.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "FIFO is always a bad scheduling policy." | It performs perfectly reasonably when job lengths are roughly equal (as in the first worked example) — its weakness specifically appears when job lengths vary significantly, producing the convoy effect. |
| "The convoy effect is caused by too many jobs arriving at once." | It's caused specifically by short jobs being stuck behind one (or more) long jobs under a non-preemptive, strict-arrival-order policy — it has nothing to do with the total number of jobs, only their relative lengths and ordering. |

## Interview Questions

1. **Q: How does FIFO scheduling work?**
   A: It runs jobs strictly in the order they arrive, running each one to full, non-preemptive completion before starting the next — no job can be interrupted once it begins running.

2. **Q: What is the convoy effect, and when does it occur?**
   A: When one or more short jobs get stuck waiting behind a long-running job under a non-preemptive, arrival-order-based policy like FIFO — it occurs whenever job lengths vary significantly and a long job happens to arrive before (or at the same time as) shorter ones.

3. **Q: Why does the convoy effect specifically hurt average turnaround time?**
   A: Because the short jobs, which could individually finish almost instantly, are instead forced to wait for the entire duration of the long job first — inflating their own turnaround times far more than necessary and dragging down the average across all jobs.

## Summary

- FIFO runs jobs strictly in arrival order, to full completion, with no preemption.
- It performs well when job lengths are roughly uniform, but suffers badly from the convoy effect when short jobs get stuck behind long ones.
- The convoy effect directly motivates the next topic's policy: run the shortest job first, regardless of arrival order, to minimize average turnaround time.
