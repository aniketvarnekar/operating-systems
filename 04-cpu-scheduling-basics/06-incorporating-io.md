# Incorporating I/O

## Learning Objectives

By the end of this section you should be able to:
- Explain why treating jobs as purely CPU-bound is an unrealistic simplification
- Explain how a scheduler should react when a running job issues an I/O request
- Explain the concept of overlapping I/O and computation to keep the CPU busy

## Prerequisites

- Topic 4 (Shortest Time-to-Completion First)
- Topic 5 (Round Robin)
- Module 02, Topic 2 (Process States and Lifecycle) — specifically, the Blocked state

## Motivation

Every example so far in this module quietly assumed jobs do nothing but compute — they never read a file, never wait on a network response, never pause for anything except the scheduler's own decisions. Real jobs constantly issue I/O requests, and a scheduler that ignores this leaves enormous amounts of CPU time sitting idle unnecessarily. This topic relaxes that simplification and shows what changes.

## Problem Statement

Suppose a job's actual behavior looks like this: compute for a while, then issue a disk read request, then (once the data arrives) compute a bit more using that data, then issue another disk read, and so on — alternating between CPU work and I/O waits. Recall from Module 02, Topic 2 that issuing an I/O request moves a process from Running to **Blocked** — it cannot usefully use the CPU again until that I/O completes. If the scheduler treated this job as one single, uninterruptible "100 seconds of work" (as every earlier example in this module implicitly assumed), it would badly misrepresent what's actually happening — and worse, it would leave the CPU sitting completely idle during every one of that job's I/O waits, exactly the problem multiprogramming was invented to solve (Module 01, Topic 2).

## Concept

### Treating Jobs as a Sequence of CPU and I/O Bursts

> A realistic job is better modeled not as one solid block of CPU time, but as an alternating sequence of **CPU bursts** (periods of pure computation) and **I/O bursts** (periods where the process is Blocked, waiting on a slow I/O device). A scheduler's job is to make good decisions at the *boundary* of each CPU burst — not to treat the whole job as one indivisible unit.

### What the Scheduler Should Do When a Job Blocks on I/O

The instant a running job issues an I/O request and transitions to Blocked (Module 02, Topic 2), it can no longer usefully use the CPU — so a well-designed scheduler should **immediately schedule a different, ready job**, rather than leaving the CPU idle while the first job's I/O completes in the background. This is a direct, practical application of the CPU-virtualization goal from Module 01: don't waste the shared, scarce CPU resource on a process that currently has nothing useful for it to do.

### Applying This to STCF: Treating Each CPU Burst as Its Own "Job"

Once you think in terms of CPU bursts rather than whole jobs, a scheduling policy like STCF (Topic 4) can be applied *within* a single job's lifetime, not just across separate jobs: each time a job returns from an I/O wait with a new, short CPU burst to perform, it competes for the CPU exactly as if it were a new, short "job" arriving fresh — and STCF's normal preference for shorter work applies naturally. A job that does frequent, short bursts of computation between I/O waits (typical of interactive or I/O-heavy programs) ends up looking, from the scheduler's perspective, like a series of many short jobs — which STCF (and Round Robin) already handle well — rather than one long one.

### Overlapping I/O and Computation

The deeper goal this enables: instead of the CPU sitting idle while one process waits on a slow disk or network operation, the scheduler runs a *different* ready process during that wait — **overlapping** one process's I/O time with another process's useful CPU time. This is precisely the mechanism that lets a modern system with many I/O-heavy processes (web browsers waiting on network responses, editors waiting on disk reads) still keep the CPU highly utilized overall, rather than idling every time any single process blocks.

## Internal Working (Preview)

```
 Naive view (this module's earlier examples): one solid job

   Job A: |███████████████████████████████████████| 100s CPU

 Realistic view: alternating CPU and I/O bursts

   Job A: |CPU: 10s|--- I/O wait: 20s ---|CPU: 5s|--- I/O wait: 15s ---|CPU: 10s|...

 What a GOOD scheduler does during Job A's I/O waits:

   Job A: |CPU:10s|--- I/O (Blocked) ---|CPU:5s|--- I/O (Blocked) ---|CPU:10s|
   Job B:          |CPU: 15s (runs HERE, while A is blocked, keeping CPU busy)|
```

Without this overlap, the CPU would sit completely idle during both of Job A's I/O waits — wasted capacity that Job B's work could have used productively.

## Real-World Analogy

Think of a chef (the CPU) working on multiple dishes (jobs) at once in a busy kitchen. One dish needs to simmer, untouched, for twenty minutes (an I/O burst) — a naive chef would just stand and stare at that pot for the full twenty minutes, doing nothing else (the CPU sitting idle). A skilled chef instead immediately turns to chopping vegetables for a *different* dish the instant the first one starts simmering (scheduling a different ready job), returning to the simmering dish only once it needs attention again — keeping themselves continuously busy and productive throughout the whole shift, rather than idling through every single waiting period.

## Why This Design Is Necessary

Ignoring I/O bursts and treating every job as one solid block of CPU time isn't just an inaccurate model — it actively squanders the exact resource (CPU time) that scheduling exists to manage well. Multiprogramming's entire premise (Module 01, Topic 2), which every scheduling policy in this module ultimately builds on, is precisely "run something else useful while one process is stuck waiting on I/O" — this topic simply makes that principle explicit and shows how it integrates with the specific policies (STCF, Round Robin) covered earlier.

## Advantages of Modeling Jobs This Way

- **Dramatically better CPU utilization** — the CPU is kept busy running some ready process almost continuously, instead of idling through every I/O wait.
- **Naturally favors interactive/I/O-heavy jobs under STCF** — since their individual CPU bursts tend to be short, they're treated favorably by a shortest-burst-first policy without needing any special-case logic.

## Disadvantages / Complexities

- **Requires the scheduler to make many more, smaller decisions** — once per CPU burst rather than once per whole job — increasing the frequency of scheduling decisions (and their associated overhead, Module 03, Topic 4) compared to the simplified whole-job model used earlier in this module.
- **Burst lengths are not perfectly predictable in advance**, exactly like whole-job lengths under SJF/STCF (Topic 3–4) — real schedulers typically estimate upcoming CPU burst length based on a process's recent historical behavior, which is an imperfect prediction, not a certainty.

## Best Practices

- When reasoning about how well a real system utilizes its CPU, always ask whether ready processes are available to fill in during another process's I/O waits — a system with only one runnable process at a time can't benefit from this overlap at all, no matter how good its scheduler is.
- Recognize that "the scheduler favors short jobs" (Topic 3–4) and "the scheduler favors jobs with short CPU bursts" (this topic) are really the same underlying principle, just applied at two different granularities — a whole job, versus one CPU burst within a job's lifetime.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A job's total CPU need is just one fixed number the scheduler considers once." | Realistic jobs alternate between CPU bursts and I/O waits; a good scheduler makes a fresh decision at the boundary of each new CPU burst, not just once per whole job. |
| "When a process blocks on I/O, the CPU should just wait for it, since that process still 'owns' its turn." | The CPU should immediately be given to a different ready process instead — leaving it idle while one process is Blocked wastes the exact resource scheduling is meant to manage efficiently, directly undermining the goals of multiprogramming (Module 01, Topic 2). |

## Interview Questions

1. **Q: Why is modeling a job as "alternating CPU and I/O bursts" more realistic than treating it as one solid block of CPU time?**
   A: Because real programs routinely pause to wait on slow I/O operations (disk reads, network responses), during which they can't use the CPU at all (Module 02, Topic 2's Blocked state) — modeling this explicitly lets the scheduler make a fresh, useful decision at each burst boundary instead of one oversimplified decision for the whole job.

2. **Q: What should a scheduler do the instant a running process blocks on I/O?**
   A: Immediately schedule a different ready process, rather than leaving the CPU idle during that process's I/O wait — this is the direct, practical continuation of multiprogramming's core goal of keeping the CPU busy.

3. **Q: How does treating jobs as CPU bursts interact naturally with STCF?**
   A: Each time a process returns from an I/O wait with a new CPU burst, it competes for the CPU as if it were a fresh, short "job" — since I/O-heavy or interactive processes tend to have short CPU bursts, STCF's preference for shorter work naturally favors them without any special-case logic being needed.

## Summary

- Real jobs alternate between CPU bursts and I/O bursts, not one solid, uninterrupted block of computation.
- A good scheduler immediately runs a different ready process the instant the current one blocks on I/O, keeping the CPU busy — overlapping one process's I/O wait with another's useful computation.
- Applying policies like STCF at the level of individual CPU bursts (rather than whole jobs) naturally favors I/O-heavy, interactive-style processes, without requiring any special-case handling.
- This closes out the module's core scheduling ideas — the module summary ties FIFO, SJF, STCF, and Round Robin together against the metrics from Topic 1, now including this I/O-aware refinement.
