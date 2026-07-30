# Round Robin (RR)

## Learning Objectives

By the end of this section you should be able to:
- Describe how Round Robin scheduling works, including the role of the time slice
- Calculate response time for a set of jobs under Round Robin, and contrast it with STCF
- Explain the trade-off between time-slice length and context-switching overhead

## Prerequisites

- Topic 4 (Shortest Time-to-Completion First)
- Module 03, Topic 4 (The Context Switch)

## Motivation

Topic 4 ended with a clear, unresolved tension: STCF is optimal for turnaround time but can leave a job waiting a long time before its very first turn on the CPU — unacceptable for an interactive user who expects near-immediate feedback. Round Robin is the policy that deliberately makes the opposite trade-off, optimizing specifically for response time.

## Problem Statement

Imagine three jobs of equal length, each needing 15 seconds of CPU time, all arriving at time 0. Under STCF (or SJF — they behave identically when jobs are equal length), the scheduler runs one job to completion before even starting the next:

```
 Time:    0                15               30               45
          │───────A───────│───────B───────│───────C───────│
```

Job C doesn't get *any* CPU time at all until t=30 — a 30-second wait just to be first acknowledged, even though its own total work only takes 15 seconds. For an interactive job (imagine each of these is a user's command, not a batch job), this is a poor response time, regardless of how good the average turnaround time looks.

## Concept

### The Policy

> **Round Robin (RR)** runs each ready job for a fixed, small unit of time called a **time slice** (or **scheduling quantum**), then moves on to the next ready job in a repeating cycle — giving every job a turn again and again, in short bursts, rather than running any one job to completion before considering others.

### Worked Example (Same Three Equal Jobs, Time Slice = 5)

```
 Time:    0    5    10   15   20   25   30   35   40   45
          │─A─│─B─│─C─│─A─│─B─│─C─│─A─│─B─│─C─│─A─│─B─│─C─│
```

(Each job needs 15 seconds total, so 3 rounds of 5-second slices each.)

- Every job gets its **first** turn within the first 15 seconds (A at t=0, B at t=5, C at t=10) — compare this to STCF's worst case above, where C waited a full 30 seconds for its first turn.
- **Average response time is dramatically better** than STCF's for this scenario, precisely because no job has to wait for others to *fully finish* before getting even a small slice of attention.
- **Average turnaround time, however, gets worse**: each job now finishes later than it would have under STCF, because time is being split between jobs instead of dedicated to finishing one at a time.

This is the direct, deliberate trade-off Round Robin makes: it sacrifices some average turnaround time specifically to buy dramatically better response time.

### The Time Slice Length Trade-off

The length of the time slice is the single most important tuning parameter for Round Robin, and it creates a direct tension:

- **Too short a time slice**: response time improves further, but the *overhead* of constantly context-switching (Module 03, Topic 4) between jobs — each switch costing real, non-productive time — starts to dominate, wasting CPU time on switching rather than useful work.
- **Too long a time slice**: context-switching overhead shrinks, but Round Robin starts behaving more and more like FIFO (Topic 2), since jobs effectively run for long, uninterrupted stretches again — losing the response-time benefit that was the entire point.

> The general principle: the time slice should be **long relative to the cost of a context switch** (so switching overhead stays a small fraction of total time), but still **short enough** to deliver good response time for interactive use.

## Internal Working (Preview)

```
 Round Robin's core loop:

   maintain a queue of Ready jobs
   loop:
       take the job at the front of the queue
       run it for exactly one time slice (or until it
       finishes/blocks, if sooner)
       if it still has work left, put it at the BACK
       of the queue
       repeat
```

Note that unlike STCF, Round Robin needs **no knowledge of job length at all** — it treats every ready job identically, regardless of how long it will ultimately run, which is a meaningful practical advantage over STCF's dependency on (often imperfectly) known or estimated job lengths.

## Real-World Analogy

Think of Round Robin like a group project meeting where the facilitator gives every participant exactly two minutes to speak, in a repeating cycle, rather than letting the first speaker talk for as long as they want before moving to the next person. Everyone gets to say *something* early on (good response time), even if it means the meeting as a whole takes longer to fully wrap up than if each person had simply spoken until they were completely done, one at a time (worse total turnaround/completion time for the meeting overall).

## Why Round Robin Is Designed This Way

Round Robin exists precisely because interactive computing (Module 01, Topic 2's time-sharing era) fundamentally cares more about "does this feel responsive right now" than "what's the absolute fastest average completion time across all jobs." STCF's optimal-turnaround-time guarantee is worthless to a user who is sitting and staring at an unresponsive terminal, waiting for their very first sign of progress — Round Robin directly targets that experience instead, at a deliberate, accepted cost to raw average completion time.

## Advantages of Round Robin

- **Excellent response time** — every job gets attention quickly, regardless of its total length, making it well-suited to interactive workloads.
- **No job-length information required** — unlike SJF/STCF, Round Robin doesn't need to know or estimate how long any job will take in advance.
- **No starvation** — every ready job is guaranteed to get a turn within one full cycle of the queue, so no job waits indefinitely.

## Disadvantages of Round Robin

- **Worse average turnaround time** than STCF, particularly noticeable for a mix of long and short jobs, since even short jobs must wait through a full round for every other job before continuing.
- **Sensitive to time-slice length** — too short wastes CPU time on excessive context-switching overhead (Module 03, Topic 4); too long degrades toward FIFO-like behavior, eroding the response-time benefit.

## Best Practices

- When designing or evaluating a scheduler for an interactive system (desktop OS, shells, GUIs), favor Round-Robin-style time-slicing, and specifically examine whether the time-slice length is well-tuned relative to context-switch cost.
- When designing or evaluating a scheduler for a purely batch workload with no interactive component, favor STCF-style scheduling instead — Round Robin's response-time advantage is wasted if nothing is watching the jobs run in real time.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A shorter time slice is always better because it improves response time." | An excessively short time slice increases the *frequency* of context switches, and each switch has a real, non-zero cost (Module 03, Topic 4) — past a certain point, the overhead of switching itself starts to dominate, wasting CPU time rather than doing useful work. |
| "Round Robin is simply a worse version of STCF." | They optimize for different metrics entirely — STCF is provably better for average turnaround time, but Round Robin is dramatically better for response time and requires no advance knowledge of job length; which is "better" depends entirely on the workload's actual needs (Topic 1). |

## Interview Questions

1. **Q: How does Round Robin scheduling work?**
   A: Each ready job runs for a fixed time slice, then is placed at the back of the ready queue, cycling through all ready jobs repeatedly, rather than running any single job to completion before considering others.

2. **Q: What is the fundamental trade-off Round Robin makes compared to STCF?**
   A: Round Robin dramatically improves response time (every job gets attention quickly) at the cost of worse average turnaround time (jobs individually take longer to fully finish, since CPU time is divided among all ready jobs rather than dedicated to finishing the shortest one first).

3. **Q: What happens if the time slice in Round Robin is set too short?**
   A: Context-switching overhead (Module 03, Topic 4) starts to dominate — the CPU spends a significant, wasted fraction of its time performing switches between jobs rather than doing useful work for any of them.

4. **Q: Does Round Robin need to know how long each job will run, unlike SJF/STCF?**
   A: No — Round Robin treats every ready job identically regardless of its total length, which is a real practical advantage over SJF/STCF's dependency on knowing or estimating job lengths in advance.

## Summary

- Round Robin runs each ready job for a fixed time slice in a repeating cycle, rather than running jobs to completion one at a time.
- It dramatically improves response time compared to STCF/SJF, at a real cost to average turnaround time.
- The time-slice length is a critical tuning parameter: too short wastes time on context-switching overhead; too long erodes toward FIFO-like behavior.
- Unlike SJF/STCF, Round Robin requires no advance knowledge of job length, making it a practical, general-purpose choice — especially for interactive workloads.
- The next topic relaxes a simplifying assumption used throughout this whole module (that jobs are purely CPU-bound, with no I/O), showing how a scheduler should account for jobs that also need to wait on slow I/O operations.
