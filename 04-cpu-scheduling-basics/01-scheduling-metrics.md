# Scheduling Metrics

## Learning Objectives

By the end of this section you should be able to:
- Define turnaround time and response time precisely, with formulas
- Explain why a scheduler optimized purely for one metric can perform badly on the other
- Explain what fairness means in a scheduling context, and why it can conflict with pure performance

## Prerequisites

- Module 03 (Direct Execution) in full

## Motivation

Before judging whether one scheduling policy is "better" than another, you need precise, measurable definitions of "better" — otherwise every comparison in Topics 2–5 is just vague intuition. This topic gives you the exact vocabulary the rest of the module relies on.

## Problem Statement

Suppose you're comparing two different scheduling policies, and each one produces a different order in which three jobs finish. How do you decide, objectively, which order was "better"? "Better" clearly depends on what you actually care about — a batch data-processing job and an interactive text editor have very different ideas of what a good experience looks like — so you need more than one metric, and you need to know precisely how each is calculated.

## Concept

### Turnaround Time

> **Turnaround time** is the total time from when a job arrives (is ready to run) until it completes: `T_turnaround = T_completion − T_arrival`.

This metric captures overall throughput-style performance — it's what a batch job (a script that just needs to finish, with no one watching it run) cares about most. A scheduler that minimizes average turnaround time gets jobs *done* as quickly as possible on average, even if it means some individual jobs wait a long time before even starting.

### Response Time

> **Response time** is the time from when a job arrives until it *first* gets scheduled onto the CPU (even briefly) for the first time: `T_response = T_first_run − T_arrival`.

This metric captures interactivity — it's what matters for a human sitting at a keyboard, waiting to see *any* sign their input is being handled, even if the underlying task takes much longer to fully finish. A scheduler can have excellent response time (you see something happen almost instantly) while still having mediocre turnaround time (the job doesn't actually *finish* for a while) — these two metrics are measuring genuinely different things, and Topics 4–5 will show policies that excel at one specifically at the cost of the other.

### Fairness

> **Fairness**, informally, means giving every job a reasonably equal share of the CPU over time, rather than letting some jobs finish very quickly while others wait an excessively long time.

Fairness often directly conflicts with pure turnaround-time optimization: as Topics 3–4 will show, deliberately favoring short jobs over long ones improves *average* turnaround time, but can make a long job wait a very long time in an extreme case — great on average, unfair to that one long job. Every scheduling policy in this module makes an implicit or explicit trade-off between raw average performance and fairness across individual jobs.

## Internal Working (Preview)

```
   Job arrives          Job first runs           Job completes
       │                       │                        │
       ├───────────────────────┤                        │
       │   response time        │                        │
       │                                                 │
       ├─────────────────────────────────────────────────┤
       │                turnaround time                   │
```

A job can have a short response time (it starts quickly) but a long turnaround time (it takes a long time to actually finish, perhaps because it keeps getting preempted to let other jobs run) — the two metrics are computed from different endpoints and don't move together.

## Real-World Analogy

Think of a restaurant. **Response time** is how quickly a server acknowledges you after you sit down — even just "I'll be right with you," not your food arriving. **Turnaround time** is how long from sitting down until you've actually finished your entire meal and left. A restaurant can have excellent response time (a server greets every table within thirty seconds) while still having mediocre turnaround time (the kitchen is slow, so full meals take a long time) — these are simply different measurements of different parts of the experience, and a restaurant optimizing purely for "greet fast" might not be optimizing for "serve fast," and vice versa.

## Why These Specific Metrics Are Used

Turnaround time and response time were chosen because they map directly onto the two dominant real-world use cases for a computer: **batch-style workloads** (where nobody is watching a job run, and only the total completion time matters — directly descended from Module 01, Topic 2's batch-processing era) and **interactive workloads** (where a human is actively waiting for visible feedback — directly descended from Module 01, Topic 2's time-sharing era). Fairness is included because a scheduler that only optimizes raw average metrics can produce individually unacceptable outcomes (one job waiting far too long), even while its averages look excellent.

## Advantages of Having Precise Metrics

- **Objective comparison** — you can now say precisely *why* one scheduling policy outperforms another for a specific goal, instead of relying on vague impressions.
- **Exposes real trade-offs** — as Topics 2–5 will demonstrate concretely, no single policy in this module maximizes both turnaround time and response time simultaneously; precise metrics are what let you see and quantify that trade-off instead of missing it.

## Disadvantages / Limitations

- **No single metric captures everything that matters** — a policy can look excellent on average turnaround time while treating individual jobs very unfairly, which average-based metrics alone can hide unless you separately examine fairness.
- **Real workloads mix job types** — a real system runs both batch and interactive jobs simultaneously, and a policy optimized purely for one metric can perform poorly for workloads that actually care more about the other (a running theme through the rest of this module).

## Best Practices

- When comparing scheduling policies, always ask "better by which metric?" before accepting a claim that one policy is simply "better" than another — as this module will repeatedly show, a policy can be strictly better on one metric and strictly worse on another.
- Consider both the *average* value of a metric across all jobs, and the *worst-case* value for any individual job — a policy can have an excellent average while still treating one unlucky job very unfairly.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Turnaround time and response time measure basically the same thing." | Turnaround time measures total time to completion; response time measures only the time until a job first gets any CPU attention at all. A job can score very differently on each. |
| "A scheduler that optimizes average turnaround time is automatically 'the best' scheduler." | It may achieve poor response time (bad for interactive use) or poor fairness (one job waiting excessively long) even while its average turnaround time looks great — "best" always depends on which metric, and whose experience, you're optimizing for. |

## Interview Questions

1. **Q: What's the difference between turnaround time and response time?**
   A: Turnaround time is the total time from a job's arrival until it fully completes. Response time is the time from arrival until the job is first given any CPU time at all, even briefly — a much shorter, interactivity-focused measurement.

2. **Q: Why can't a scheduler simply be judged as "good" or "bad" in an absolute sense?**
   A: Because "good" depends on which metric matters for the workload — a policy can excel at average turnaround time (great for batch jobs) while performing poorly on response time (bad for interactive use), or vice versa; there's no policy in this module that maximizes both simultaneously.

3. **Q: Why does fairness matter as a separate concern from average turnaround time?**
   A: Because a policy can achieve an excellent average turnaround time while still treating one specific job very unfairly (making it wait an extremely long time) — averages alone can hide poor outcomes for individual jobs.

## Summary

- Turnaround time measures total time from arrival to completion; response time measures time from arrival to first being scheduled at all.
- These two metrics can move independently — excellent response time doesn't imply excellent turnaround time, and vice versa.
- Fairness is a separate concern from raw average performance — a policy can look great on average while treating specific jobs unfairly.
- The next topics walk through FIFO, SJF, STCF, and Round Robin as a chain of policies, each one explicitly trading off between these metrics.
