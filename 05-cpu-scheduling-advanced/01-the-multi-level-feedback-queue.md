# The Multi-Level Feedback Queue (MLFQ)

## Learning Objectives

By the end of this section you should be able to:
- Explain the core problem MLFQ solves that STCF and Round Robin, individually, cannot
- State MLFQ's basic rules and trace through a simple example
- Explain how MLFQ approximates STCF's turnaround-time benefits and Round Robin's response-time benefits simultaneously

## Prerequisites

- Module 04, Topic 4 (Shortest Time-to-Completion First)
- Module 04, Topic 5 (Round Robin)

## Motivation

Module 04 left a real, practical gap: STCF gives excellent turnaround time but requires knowing job lengths in advance — information a real OS almost never has when a process first starts. Round Robin needs no such information but sacrifices turnaround time across the board. MLFQ is the classic answer to "can we get the best of both, without requiring the OS to be told job lengths ahead of time?"

## Problem Statement

An OS scheduler must decide how to treat a brand-new process the very first instant it starts running — at that moment, the OS has no idea whether this process will be a short, interactive command that finishes in milliseconds, or a long-running batch computation that will need the CPU for hours. Guessing wrong has real costs: treating every process as if it might be long (like plain Round Robin) sacrifices the excellent turnaround time STCF could have given short jobs; but STCF's optimal behavior specifically requires knowing job length in advance — information that, for a brand-new process, simply doesn't exist yet.

Is there a way to get STCF-like treatment for genuinely short jobs and Round-Robin-like fairness for longer ones, without ever being told in advance which category a given process belongs to?

## Concept

### The Core Idea: Learn From Observed Behavior

> The **Multi-Level Feedback Queue (MLFQ)** maintains several distinct priority queues. It does not require any job length to be known in advance. Instead, it **observes how a process actually behaves as it runs**, and adjusts that process's priority over time based on that observed behavior — using history as a substitute for advance knowledge.

The key insight: a process's *past* behavior (has it been using long, uninterrupted bursts of CPU time, or short bursts before yielding/blocking) is a reasonably good predictor of its *future* behavior, even without knowing anything about it in advance.

### The Basic Rules

1. There are multiple queues, each assigned a different **priority level**. Higher-priority queues are always checked first — if there's a ready job in a higher-priority queue, it's chosen over any job in a lower-priority queue.
2. Jobs within the *same* priority queue are scheduled Round-Robin-style (Module 04, Topic 5) among each other.
3. **A new job enters at the highest priority level** — it is initially given the benefit of the doubt, treated as if it might be a short, interactive job.
4. **If a job uses its entire allotted time slice without voluntarily giving up the CPU (i.e., it behaves like a long-running, CPU-bound job), its priority is lowered** — moved down to the next lower queue.
5. **If a job gives up the CPU voluntarily before its time slice ends** (typically because it blocked on I/O, Module 04, Topic 6, behaving like a short, interactive burst), **it stays at the same priority level.**

### Why This Approximates STCF and Round Robin Simultaneously

- A genuinely short, interactive job (a quick command, a program that mostly waits on user input or I/O) will typically finish, or voluntarily yield, well before using up its time slice at the highest priority level — under Rule 5, it stays at high priority throughout its (short) life, getting fast, STCF-like treatment without the OS ever needing to know in advance that it was short.
- A long-running, CPU-bound job will keep using its entire time slice every round — under Rule 4, it gets progressively demoted to lower and lower priority levels over time, converging toward Round-Robin-style treatment among other similarly long-running jobs, while no longer competing directly against genuinely short jobs for the highest-priority slots.

This is the elegant core of MLFQ: **priority itself becomes the OS's evolving estimate of "how long is this job likely to still need the CPU," inferred purely from its own observed, recent behavior** — no advance knowledge required at all.

## Internal Working (Preview)

```
 Priority Level 1 (highest) ──► [ new jobs start here; short/interactive jobs stay here ]
 Priority Level 2            ──► [ jobs that used a full time slice once get demoted here ]
 Priority Level 3 (lowest)   ──► [ long-running, CPU-bound jobs converge here over time ]

 Scheduling rule: always run from the highest non-empty queue;
                  within a queue, Round Robin (Module 04, Topic 5)

 A job's priority moves DOWN one level each time it uses
 a full time slice without yielding. It's job's priority
 STAYS THE SAME if it yields voluntarily before its slice ends.
```

## Real-World Analogy

Think of a customer-service queue at a company that doesn't ask customers in advance "how complicated is your issue" (since customers themselves often don't know) — instead, it initially treats every new caller as a potential "quick question" and puts them in the fast lane. If a call goes past a certain time threshold without resolving (a full time slice used up), that customer is automatically moved to a slower, lower-priority queue for their *next* call, on the theory that a call this long probably indicates a genuinely complex issue, not a quick one. A customer whose call wraps up quickly (voluntarily hangs up, satisfied, before the threshold) stays in the fast lane for next time — the system is learning each customer's likely complexity purely from their own observed call history, without ever being told in advance.

## Why MLFQ Is Designed This Way

The alternative — requiring accurate advance knowledge of job length, as SJF/STCF do (Module 04, Topics 3–4) — is simply not available in most real systems; a process's actual runtime is rarely known the moment it starts. MLFQ's insight (use recent observed behavior as a proxy for future behavior) sidesteps this entirely, adapting dynamically instead of requiring information the OS doesn't have. This general "learn from observed history instead of requiring advance knowledge" pattern reappears in other areas of OS design as well (e.g., predicting future memory access patterns, Module 11).

## Advantages of MLFQ (Basic Rules)

- **No advance job-length knowledge required** — a major practical advantage over SJF/STCF.
- **Approximates STCF's turnaround-time benefit for short jobs**, and **Round Robin's response-time benefit** for genuinely interactive jobs, by adapting priority based on observed behavior.

## Disadvantages of the Basic Rules (Previewed)

- **Long-term starvation risk**: if there's a constant stream of new, high-priority (interactive) jobs arriving, a job that's been demoted to a low priority level could, in principle, wait a very long time, since higher-priority queues are always checked first.
- **Vulnerable to gaming**: a cleverly (or maliciously) written process could deliberately issue a trivial, near-instant I/O operation just before its time slice would otherwise expire, repeatedly triggering Rule 5 ("yielded voluntarily, stays at the same priority") to remain at high priority indefinitely, without ever actually being a genuinely short job.

Both of these specific weaknesses, and the concrete rule refinements that address them, are the subject of the next topic.

## Best Practices

- When explaining MLFQ, always lead with the "learns from observed behavior instead of requiring advance knowledge" framing — that's the single idea that makes every specific rule make sense, rather than seeming like arbitrary bookkeeping.
- Keep Rule 4 and Rule 5 as a matched pair in your mental model: "used full slice → demote" and "yielded early → stay" are two sides of the exact same behavioral signal.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "MLFQ requires knowing whether a job is 'short' or 'long' before it starts, just like SJF." | This is precisely what MLFQ avoids — every job starts at the highest priority regardless of its true nature, and priority only changes afterward, based on how the job actually behaves once running. |
| "A job's priority in MLFQ is fixed once assigned." | Priority is continuously re-evaluated based on ongoing behavior — a job that starts behaving differently (e.g., a long batch job that suddenly starts making frequent short I/O calls) can have its priority treatment change accordingly over time (refined further in Topic 2). |

## Interview Questions

1. **Q: What problem does MLFQ solve that STCF cannot, in practice?**
   A: STCF requires knowing (or accurately estimating) a job's total length in advance to make optimal scheduling decisions — information a real OS typically doesn't have for a brand-new process. MLFQ sidesteps this by observing how a job actually behaves as it runs and adjusting its priority accordingly, requiring no advance knowledge at all.

2. **Q: What are MLFQ's core rules regarding priority changes?**
   A: A new job starts at the highest priority. If a job uses its entire time slice without voluntarily yielding, its priority is lowered. If it voluntarily yields the CPU before its time slice ends (typically due to I/O), it stays at the same priority level.

3. **Q: How does MLFQ approximate both STCF's and Round Robin's benefits simultaneously?**
   A: Genuinely short or interactive jobs tend to yield before their time slice ends, so they stay at high priority throughout their short life — getting fast, STCF-like treatment. Long-running, CPU-bound jobs repeatedly use their full time slice and get progressively demoted, converging toward Round-Robin-style treatment among similarly long jobs, without competing directly against short jobs for top priority.

## Summary

- MLFQ maintains multiple priority queues and adjusts a job's priority based on its observed behavior, rather than requiring job length to be known in advance.
- A new job starts at the highest priority; using a full time slice without yielding demotes it; voluntarily yielding before the slice ends keeps it at the same priority.
- This lets MLFQ approximate STCF's turnaround-time benefit for short/interactive jobs and Round Robin's response-time benefit, simultaneously, purely from observed history.
- The next topic covers two real problems with these basic rules — long-term starvation and gaming — and the specific refinements real schedulers add to address them.
