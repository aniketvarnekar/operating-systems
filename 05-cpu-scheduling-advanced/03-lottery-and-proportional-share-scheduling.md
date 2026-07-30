# Lottery and Proportional-Share Scheduling

## Learning Objectives

By the end of this section you should be able to:
- Explain the core idea of proportional-share scheduling and how it differs from every policy in Module 04
- Explain how lottery scheduling implements proportional share using tickets and randomness
- Explain the trade-off between lottery scheduling's simplicity and its short-term unfairness, and how stride scheduling addresses it

## Prerequisites

- Module 04, Topic 1 (Scheduling Metrics)
- Topic 1 (The Multi-Level Feedback Queue) — helpful context, not required

## Motivation

Every policy so far — FIFO, SJF, STCF, Round Robin, MLFQ — has been judged by turnaround time, response time, or (implicitly) rough fairness. This topic introduces a different goal entirely: guaranteeing each process a specific, deliberately-chosen *proportion* of total CPU time (e.g., "process A should get 50% of the CPU, process B should get 30%, process C should get 20%") — a genuinely different question from "which job finishes soonest" or "which job gets first attention."

## Problem Statement

Imagine you're running a system where you want explicit, guaranteed control over *how much* of the CPU each process gets, relative to the others — not just "run in some fair-ish order," but "guarantee process A consistently gets roughly twice as much CPU time as process B, over any reasonably long period." None of Module 04's policies, nor MLFQ, are designed to make this specific kind of proportional guarantee directly — they optimize for turnaround/response time or approximate fairness, but don't let you dial in exact relative shares.

## Concept

### Proportional-Share Scheduling

> **Proportional-share scheduling** aims to give each process a specific, guaranteed *percentage* (share) of the CPU, rather than optimizing for turnaround time or response time directly. It's a fundamentally different scheduling goal from everything in Module 04.

### Lottery Scheduling

> **Lottery scheduling** implements proportional share using **tickets** and randomness: each process is assigned some number of tickets, representing its intended share of the CPU. At each scheduling decision, the scheduler picks a random winning ticket (uniformly, across the total number of tickets held by all ready processes), and whichever process holds that ticket gets to run next.

A process holding more tickets has a proportionally higher probability of winning any given lottery — a process with twice as many tickets as another will, over many repeated drawings, win roughly twice as often, converging toward its intended proportional share, purely as a statistical consequence of holding more tickets in the pool.

### Worked Example

Suppose three processes hold tickets as follows: A has 50 tickets, B has 30 tickets, C has 20 tickets (total = 100 tickets). At each scheduling decision, the scheduler draws a uniformly random number from 1 to 100:

- A number from 1–50 → A wins (50% probability, matching its 50 tickets out of 100)
- A number from 51–80 → B wins (30% probability)
- A number from 81–100 → C wins (20% probability)

Over a large number of drawings, A will run roughly 50% of the time, B roughly 30%, and C roughly 20% — proportional to their ticket counts, exactly as intended.

### Why Randomness, of All Things?

Randomness might seem like an odd tool for something as important as fair resource allocation, but it has a specific, genuine engineering advantage: it requires almost no bookkeeping or long-term state. Unlike MLFQ, which must carefully track each process's detailed history (Topics 1–2), a lottery scheduler only needs each process's current ticket count and a random number generator — no history of past decisions is needed to make the *next* one, since each drawing is independent, yet the proportional guarantee still emerges correctly over time as a simple consequence of probability.

### The Short-Term Fairness Weakness, and Stride Scheduling

Randomness's simplicity comes with a real cost: **over any short period, actual outcomes can deviate meaningfully from the intended proportions purely by chance** — much like flipping a fair coin twice doesn't guarantee exactly one heads and one tails. A process with a 20% ticket share could, by bad luck, simply not win several drawings in a row, even though its long-run average will still converge correctly.

> **Stride scheduling** achieves the same proportional-share goal **deterministically**, without any randomness at all: each process is assigned a "stride" — a value inversely proportional to its ticket count (more tickets → smaller stride) — and the scheduler always picks whichever process has the smallest current "pass value" (a running total incremented by its own stride each time it runs), guaranteeing exact, deterministic proportional shares over any interval, not just on long-run average.

Stride scheduling trades away lottery scheduling's elegant simplicity (no state needed beyond ticket counts) for deterministic short-term fairness (at the cost of tracking each process's running pass value).

## Internal Working (Preview)

```
 Lottery scheduling (probabilistic):

   Tickets:  A=50   B=30   C=20   (total = 100)
   Draw a random number 1–100 each time → winner runs
   Long-run average: A≈50%, B≈30%, C≈20%
   Short-run: can deviate meaningfully from these proportions by chance

 Stride scheduling (deterministic):

   Stride is inversely proportional to tickets (more tickets = smaller stride)
   Always run whichever process has the SMALLEST current "pass value"
   Increment that process's pass value by its own stride after it runs
   → exact, deterministic proportional shares, even over short intervals
```

## Real-World Analogy

Think of lottery scheduling like a raffle where you buy tickets in proportion to how large a prize share you want — buying 50 out of 100 total tickets gives you good odds (50%) of winning any single drawing, and if the raffle is held very many times, your total wins will converge toward roughly half of them — but on any given individual drawing, or even a short streak of drawings, you could easily go without winning at all, purely from bad luck. Stride scheduling is like a strict, pre-planned rotation schedule instead — if you're entitled to half the total time, you are mathematically guaranteed to get called roughly every other turn, deterministically, with no possibility of an unlucky short-term streak of being skipped.

## Why This Approach Is Useful

Proportional-share scheduling is the right tool specifically when the actual goal is "guarantee relative resource shares between known, competing consumers" — for example, in shared hosting or virtualization environments, where a system operator might want to guarantee one customer's virtual machine gets exactly twice the CPU share of another's, as a direct, explicit business or fairness requirement — a fundamentally different goal from "minimize average turnaround time" (Module 04) or "learn and adapt to unknown process behavior" (MLFQ, Topics 1–2).

## Advantages of Lottery/Proportional-Share Scheduling

- **Simple, explicit control over relative CPU shares** — assigning tickets directly encodes the desired proportional allocation, without needing to reason about turnaround or response time at all.
- **Minimal bookkeeping (for lottery specifically)** — no historical state is needed beyond current ticket counts; each drawing is independent.
- **Naturally resistant to certain gaming strategies** — unlike MLFQ's yield-based rules (Topic 2), there's no obvious way for a process to manipulate its own odds of winning beyond legitimately being assigned more tickets.

## Disadvantages / Trade-offs

- **Short-term unfairness under pure lottery scheduling** — actual outcomes can deviate from intended proportions over any limited window, purely due to randomness, even though long-run averages converge correctly.
- **Requires an external mechanism to assign ticket counts sensibly** — the scheduler itself doesn't decide what a "fair" number of tickets is for any given process; that's a policy decision made outside the scheduler itself.
- **Stride scheduling removes the short-term unfairness but reintroduces state-tracking overhead** (each process's running pass value), giving up lottery scheduling's stateless simplicity in exchange for determinism.

## Best Practices

- Reach for proportional-share scheduling specifically when the actual requirement is "guarantee relative resource shares between competing consumers," not as a general-purpose replacement for turnaround/response-time-oriented policies (Module 04) or adaptive, history-based policies (MLFQ, Topics 1–2).
- When short-term fairness genuinely matters (not just long-run averages), prefer stride scheduling's deterministic guarantee over lottery scheduling's probabilistic one.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Lottery scheduling guarantees each process gets exactly its proportional share every single time period." | It only guarantees the correct proportion on average, over a large number of drawings — short-term results can deviate meaningfully from the intended shares purely due to randomness. Stride scheduling is what provides a deterministic, short-term guarantee instead. |
| "Proportional-share scheduling is just another way of expressing MLFQ-style priority." | They're fundamentally different goals: MLFQ infers likely process behavior (short vs. long) from observed history to approximate good turnaround/response time; proportional-share scheduling explicitly guarantees a chosen relative percentage of CPU time, regardless of a process's actual behavior pattern. |

## Interview Questions

1. **Q: What does proportional-share scheduling guarantee that Module 04's policies (FIFO, SJF, STCF, Round Robin) do not?**
   A: A specific, chosen percentage share of total CPU time for each process, relative to others — rather than optimizing for average turnaround time, response time, or approximate fairness, which is what Module 04's policies target instead.

2. **Q: How does lottery scheduling implement proportional share?**
   A: Each process is assigned a number of tickets proportional to its intended CPU share; at each scheduling decision, a uniformly random winning ticket is drawn from the total pool, and whichever process holds it runs next — over many drawings, each process's win rate converges to its ticket share.

3. **Q: What weakness does lottery scheduling have that stride scheduling fixes?**
   A: Lottery scheduling only guarantees correct proportional shares on long-run average — short-term outcomes can deviate from the intended proportions purely by chance. Stride scheduling achieves the same proportional-share goal deterministically, guaranteeing accurate shares even over short intervals, by tracking each process's running "pass value" instead of relying on randomness.

## Summary

- Proportional-share scheduling is a fundamentally different goal from turnaround/response-time optimization: guaranteeing each process a specific, relative share of total CPU time.
- Lottery scheduling implements this via tickets and randomness — a process's win probability at each drawing matches its share of the total ticket pool, converging correctly on long-run average.
- Lottery scheduling's simplicity (minimal bookkeeping) comes at the cost of possible short-term unfairness; stride scheduling achieves the same proportional guarantee deterministically, at the cost of tracking per-process state.
- The next topic addresses an assumption every policy so far has quietly made: that there's exactly one CPU. Multi-CPU scheduling covers what changes once there's more than one core to schedule across.
