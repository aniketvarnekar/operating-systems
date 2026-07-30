# Shortest Time-to-Completion First (STCF)

## Learning Objectives

By the end of this section you should be able to:
- Describe how STCF works and how it differs from SJF in exactly one way
- Trace through a staggered-arrival example and show STCF fixing SJF's weakness
- Explain why STCF is optimal for turnaround time, and why it's still poor for response time

## Prerequisites

- Topic 3 (Shortest Job First)
- Module 03 (Direct Execution) — specifically, that preemption requires the timer interrupt mechanism from Topic 3 of that module

## Motivation

Topic 3 ended on a specific, concrete gap: SJF can't react to a shorter job arriving after a longer one has already started running, because SJF refuses to preempt. STCF closes that gap with a small, precise change — and this topic shows exactly what that change buys you, and what it still costs you.

## Problem Statement

Recall Topic 3's staggered-arrival example: job A (100 seconds) arrives at t=0 and starts running. At t=10, jobs B and C (10 seconds each) arrive, but SJF's non-preemption rule means A keeps running anyway, forcing B and C to wait needlessly — average turnaround time of 103.33, barely better than FIFO's convoy effect. What if the scheduler were allowed to interrupt A the moment a shorter job arrived?

## Concept

### The Policy

> **Shortest Time-to-Completion First (STCF)** — sometimes called Preemptive Shortest Job First (PSJF) — is exactly SJF, with one addition: it is **preemptive**. Whenever a new job arrives, the scheduler compares that new job's total length against the *remaining* time needed by the currently running job, and immediately switches the CPU to whichever one has the least time left to finish — even if that means interrupting a job that's already running.

The context switch mechanism making this interruption physically possible was covered in full in Module 03 — specifically, the timer interrupt (Module 03, Topic 3) is what allows the OS to regain the CPU at the moment a new job arrives, evaluate the new scheduling decision, and switch if warranted.

### Worked Example (Same Staggered Arrivals as Topic 3)

Job A (100s) arrives at t=0 and starts running (it's the only option). At t=10, B and C (10s each) arrive:

```
 Time:     0        10        20        30                    120
           │────A────│───B───│───C───│──────────A (resumed)──────│
                      ▲
              A is preempted here — it had 90s
              remaining, B needs only 10s, so B
              (and then C) run first
```

- A ran 0→10 (10s done, 90s remaining), gets preempted at t=10
- B runs 10→20, completes at t=20 → turnaround = 20 − 10 = 10
- C runs 20→30, completes at t=30 → turnaround = 30 − 10 = 20
- A resumes 30→120 (its remaining 90s), completes at t=120 → turnaround = 120 − 0 = 120
- **Average turnaround time = (10 + 20 + 120) / 3 = 50**

Compare this to SJF's 103.33 for the identical arrival pattern (Topic 3) — allowing preemption when a shorter job arrives recovers the same excellent average turnaround time SJF achieved only in the simultaneous-arrival case.

### STCF Is Provably Optimal for Turnaround Time

Given a set of jobs with known lengths and arrival times, **no scheduling policy can achieve a better average turnaround time than STCF** — this is a well-established theoretical result, not just an empirical observation. If you only care about minimizing average turnaround time and nothing else, STCF (or something equivalent to it) is the mathematically best possible choice.

### The Cost: Response Time

STCF's optimality is specifically for *turnaround* time (Topic 1) — it says nothing about *response* time. Consider three jobs of length 10 each, all arriving at slightly different times such that STCF always has a clear "next shortest" choice and runs each to completion before considering the next: a job that arrives while another job is deep into a long, uninterruptible run of its own remaining time could still wait a very long time before ever being scheduled even once, if that currently-running job's remaining time is still shorter than the new arrival's total length. More simply: **STCF never preempts a running job in favor of a new one unless the new one is strictly shorter than the running job's *remaining* time** — a job that's merely "reasonably short" but not the single shortest available can still sit waiting the entire time a longer job runs, giving it a poor response time despite reasonably good treatment on turnaround time overall.

## Internal Working (Preview)

```
 STCF's decision rule, evaluated every time a job arrives (or completes):

   compare: remaining time of CURRENTLY RUNNING job
        vs. total time of NEWLY ARRIVED job
   if new job is shorter → PREEMPT (context switch, Module 03, Topic 4)
   else                  → keep running the current job
```

This decision rule is the entire difference from SJF — SJF only makes this comparison once, at the moment it initially chooses a job to run, and never revisits it until that job finishes.

## Real-World Analogy

Return to the single-barber analogy from Topic 3: STCF is the same barber, but now willing to pause a longer haircut mid-way through — carefully noting exactly how much work remains — the instant someone walks in needing a much quicker trim, serving that quick trim first, then resuming the original haircut exactly where it left off. This flexibility clearly serves the group's *average* wait time well, but the customer whose long haircut kept getting paused might reasonably feel like their own individual appointment is dragging on forever, even though the barber's overall average customer wait time looks excellent.

## Why STCF's Trade-off Is Unavoidable

Prioritizing turnaround time necessarily means: whichever job can finish soonest gets the CPU, which by definition can mean other jobs — including jobs that arrived earlier, or that aren't drastically longer, just not the single shortest — are made to wait. There is no way to simultaneously guarantee "the shortest jobs always finish as fast as mathematically possible" (optimal turnaround) and "every job, however long, gets *some* CPU attention almost immediately upon arrival" (good response time) — these are fundamentally competing goals, which is exactly why Topic 5 (Round Robin) introduces a policy that makes the opposite trade-off deliberately.

## Advantages of STCF

- **Provably optimal average turnaround time** — the best possible outcome by this specific metric, given known job lengths and arrival times.
- **Directly and fully solves SJF's staggered-arrival weakness**, by allowing preemption whenever a strictly shorter job arrives.

## Disadvantages of STCF

- **Poor response time for longer jobs** — a job with a long total length can wait an extended period before ever getting its first turn on the CPU, if shorter jobs keep arriving in the meantime.
- **Still requires knowing (or estimating) job lengths in advance**, exactly like SJF — a real limitation, since actual execution time is often not known perfectly ahead of time.
- **More context-switching overhead than SJF**, since it may preempt and resume jobs multiple times rather than running each one straight through — a real cost, per Module 03, Topic 4.

## Best Practices

- When a workload is genuinely batch-oriented (no one is watching individual jobs run, only overall completion matters), STCF-style scheduling is close to ideal.
- When a workload is interactive (users expect to see prompt feedback), recognize that STCF's optimal turnaround time comes at a real response-time cost — this is exactly the scenario Topic 5's Round Robin is designed for instead.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "STCF is strictly better than SJF in every way." | STCF strictly dominates SJF on turnaround time (by adding preemption, it never does worse and often does much better) — but it does not improve response time at all, and it does incur additional context-switching overhead SJF avoids. |
| "STCF being 'optimal' means it's the best scheduler overall." | STCF is only optimal for the specific metric of average turnaround time. It can perform quite poorly on response time, which is why "optimal" always needs a metric attached to it (Topic 1). |

## Interview Questions

1. **Q: How does STCF differ from SJF?**
   A: STCF adds preemption: whenever a new job arrives, it's compared against the currently running job's *remaining* time, and the CPU switches to whichever is shorter — even mid-execution. SJF makes this comparison only once, when initially choosing a job, and never revisits it.

2. **Q: In what specific, provable sense is STCF "optimal"?**
   A: Given known job lengths and arrival times, no scheduling policy can achieve a better average turnaround time than STCF — it's the mathematically best possible policy specifically for that one metric.

3. **Q: Why does STCF perform poorly on response time despite being optimal for turnaround time?**
   A: Because STCF only preempts a running job in favor of one that's strictly shorter — a job that isn't the single shortest available can be made to wait through the current job's entire remaining execution before ever getting its first turn, producing a poor response time even while the overall average turnaround time is excellent.

## Summary

- STCF is SJF with preemption added: a newly-arrived shorter job can interrupt an already-running longer one, based on comparing the new job's length to the running job's remaining time.
- This fully closes SJF's staggered-arrival gap, and STCF is provably optimal for average turnaround time given known job lengths and arrival times.
- Its cost is poor response time for jobs that aren't the single shortest available, and additional context-switching overhead compared to non-preemptive SJF.
- The next topic, Round Robin, deliberately optimizes for the opposite metric — response time — at a real cost to turnaround time, completing the core trade-off this module is built around.
