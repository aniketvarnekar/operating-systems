# Shortest Job First (SJF)

## Learning Objectives

By the end of this section you should be able to:
- Describe how SJF scheduling works and why it fixes the convoy effect
- Calculate average turnaround time for a set of jobs under SJF
- Explain the specific new scenario (staggered arrivals) where SJF alone still performs poorly

## Prerequisites

- Topic 2 (First-In, First-Out Scheduling)

## Motivation

Topic 2 showed that FIFO's rigid arrival-order rule causes the convoy effect whenever a long job blocks shorter ones. The obvious fix: stop ordering by *arrival*, and order by *length* instead. This topic shows that fix in action — and then shows the one assumption it still silently depends on, setting up Topic 4's refinement.

## Problem Statement

Using the earlier convoy-effect example — job A needs 100 seconds, jobs B and C each need 10 seconds, all arriving at time 0 — Topic 2 showed that running them in arrival order (A, then B, then C) produced a poor average turnaround time of 110. Could simply choosing a smarter *order* — without changing anything else about how jobs run — fix this?

## Concept

### The Policy

> **Shortest Job First (SJF)** runs whichever ready job has the shortest total remaining execution time first, non-preemptively (once started, a job still runs to completion, exactly like FIFO) — the only difference from FIFO is which job is chosen first when there's a choice.

### Worked Example (Same Jobs as Topic 2)

With A(100), B(10), C(10), all arriving at t=0, SJF picks the shortest job available at each decision point:

```
 Time:     0        10        20                    120
           │───B───│───C───│──────────A──────────────│
```

- B completes at t=10 → turnaround = 10
- C completes at t=20 → turnaround = 20
- A completes at t=120 → turnaround = 120
- **Average turnaround time = (10 + 20 + 120) / 3 = 50**

Compare this to FIFO's average of 110 for the identical set of jobs (Topic 2) — simply reordering which job runs first, with no other changes, cuts the average turnaround time by more than half. This is a specific instance of a general, provable result: for a fixed batch of jobs that all arrive at the same time, running shortest-remaining-job-first minimizes average turnaround time.

### The New Problem: Staggered Arrivals

SJF's guarantee above quietly assumes all jobs are ready to choose from *at the same time*. What happens if jobs arrive at different moments? Suppose job A (100 seconds) arrives at t=0 and starts running immediately (it's the only job available). Then, at t=10, jobs B and C (10 seconds each) arrive — but SJF is non-preemptive, so A keeps running anyway, since it already started:

```
 Time:     0        10                    100      110      120
           │────────A────────────────────│───B───│───C───│
                      ▲
                B and C arrive here, but A
                already started — SJF is
                non-preemptive, so it can't
                be interrupted
```

- A completes at t=100 → turnaround = 100 − 0 = 100
- B completes at t=110 → turnaround = 110 − 10 = 100
- C completes at t=120 → turnaround = 120 − 10 = 110
- **Average turnaround time = (100 + 100 + 110) / 3 = 103.33**

This is barely better than FIFO's convoy-effect result — SJF's advantage completely evaporates the moment short jobs can arrive *after* a long job has already started, because non-preemption means SJF can't act on that new information once a job is already running.

## Internal Working (Preview)

```
 SJF, all jobs available at once:
   choose shortest of {A, B, C} → runs to completion → repeat
   Works great — full information is available at the one decision point.

 SJF, staggered arrivals:
   t=0:  only A is available → must choose A (no other option yet)
   t=10: B, C arrive, but A is already running and SJF won't preempt it
         → B and C wait needlessly, exactly like the convoy effect
```

## Real-World Analogy

Think of a single barber who, at the start of the day, looks at everyone currently waiting and serves whoever needs the shortest haircut first — genuinely smart, and fair to the group waiting at that moment. But suppose the barber has already started a long, complex styling appointment when a customer needing just a 2-minute trim walks in — the barber, following a strict "never stop a haircut once started" rule, keeps working on the long styling job anyway, making the quick-trim customer wait the entire remaining duration, even though swapping in the 2-minute job first would have served everyone far better.

## Why SJF Alone Is Not Enough

SJF assumes the scheduler always has full knowledge of every ready job's length *before* deciding what to run — genuinely true if every job arrives at the same instant, but not true in any realistic system where jobs arrive continuously over time. Because SJF is non-preemptive, it can't act on information (a newly-arrived short job) that only becomes available *after* it has already committed to running something else. This is exactly the gap Topic 4 (STCF) closes, by adding preemption to this same "prefer the shortest job" idea.

## Advantages of SJF

- **Provably optimal average turnaround time**, specifically when all jobs are available to choose from at once — no other non-preemptive ordering can do better in that specific scenario.
- **Directly eliminates the convoy effect** in the same-arrival-time case, by construction.

## Disadvantages of SJF

- **Still vulnerable to a version of the convoy effect** whenever a long job is already running and shorter jobs arrive afterward — non-preemption means the scheduler can't react to new arrivals mid-job.
- **Requires knowing (or estimating) each job's length in advance** — a real system often doesn't know exactly how long a job will run until it's done; real implementations estimate this from historical behavior, introducing its own uncertainty.
- **Potential unfairness/starvation risk** — a sufficiently long job could, in principle, be repeatedly deprioritized if shorter jobs keep arriving (this concern is developed further under STCF, Topic 4, and revisited for general fairness in Module 05).

## Best Practices

- When you see "shortest job first" mentioned without qualification, always ask whether it's the non-preemptive version (this topic) or the preemptive version (STCF, Topic 4) — they behave very differently once jobs can arrive at different times.
- Recognize that SJF's real-world weakness (staggered arrivals) is precisely a timing problem, not a flaw in "prefer shorter jobs" as an idea — which is exactly why the fix (Topic 4) keeps the same core idea and only adds preemption.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "SJF always eliminates the convoy effect completely." | It only eliminates the classic convoy effect when all competing jobs are available to choose from at the same time. If a long job is already running when short jobs arrive, SJF's non-preemption still causes those short jobs to wait unnecessarily. |
| "SJF and STCF (Topic 4) are the same algorithm." | SJF is strictly non-preemptive — once a job starts, it runs to completion. STCF adds preemption: a newly arrived shorter job can interrupt an already-running longer one. This distinction is exactly what fixes SJF's staggered-arrival weakness. |

## Interview Questions

1. **Q: How does SJF improve on FIFO?**
   A: Instead of running jobs strictly in arrival order, SJF runs whichever ready job has the shortest total execution time first — provably minimizing average turnaround time when all jobs are available to choose from at the same time, directly eliminating the classic convoy effect in that scenario.

2. **Q: Why doesn't SJF fully solve the convoy effect in a realistic system with staggered job arrivals?**
   A: Because SJF is non-preemptive — once it commits to running a job, it can't switch to a shorter job that arrives afterward, even if that new job is dramatically shorter. A long-running job that started before shorter jobs arrived still blocks them for its entire remaining duration.

3. **Q: What information does SJF require that FIFO does not?**
   A: SJF requires knowing (or estimating) each job's total execution time in advance, in order to choose the shortest one; FIFO requires no such information, since it only ever considers arrival order.

## Summary

- SJF runs the shortest available job first, non-preemptively, and is provably optimal for average turnaround time when all jobs are available simultaneously.
- It directly fixes the classic convoy effect in that scenario, cutting average turnaround time dramatically compared to FIFO for the same jobs.
- Its remaining flaw is staggered arrivals: because it's non-preemptive, a long job that's already running still blocks shorter jobs that arrive afterward, reproducing a version of the same convoy effect.
- The next topic, STCF, adds preemption to this same "prefer shorter jobs" idea, closing this gap entirely.
