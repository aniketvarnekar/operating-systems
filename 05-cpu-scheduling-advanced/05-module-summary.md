# Module 05 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Multi-Level Feedback Queue (MLFQ)** — learning a job's likely behavior from observed history instead of requiring advance knowledge, approximating both STCF and Round Robin
- [x] **MLFQ Refinements and Gaming Resistance** — periodic priority boosting (fixing starvation) and cumulative time-at-level accounting (fixing gaming)
- [x] **Lottery and Proportional-Share Scheduling** — tickets, randomness, guaranteed relative CPU shares, and stride scheduling's deterministic alternative
- [x] **Multi-CPU Scheduling** — cache affinity, single-queue vs. multi-queue designs, and load balancing

## The Big Picture

This module relaxed two assumptions Module 04 quietly made throughout: that job length is known in advance, and that there's exactly one CPU. MLFQ removes the first assumption by learning from behavior instead of requiring foreknowledge. Lottery/proportional-share scheduling reframes the goal entirely, from time-based metrics to guaranteed relative shares. Multi-CPU scheduling removes the single-CPU assumption, introducing cache affinity and load balancing as genuinely new concerns.

```
     Module 04's hidden assumptions          This module's response
     ───────────────────────────────         ───────────────────────
     "job length is known in advance"   →    MLFQ: infer it from
     (needed for SJF/STCF)                   observed behavior instead
                                              (Topics 1–2)

     "fairness ≈ turnaround/response      →  Proportional-share:
      time behaving reasonably"              explicitly guarantee a
                                              chosen relative CPU %
                                              (Topic 3)

     "there is exactly one CPU"          →   Multi-CPU scheduling:
                                              cache affinity + load
                                              balancing (Topic 4)
```

## Practical Connections

- **Why a long-running background compile doesn't permanently starve your text editor's responsiveness, even on a loaded machine** — MLFQ-style adaptive priority (Topics 1–2) keeps short, interactive bursts at high priority automatically, without any manual configuration.
- **Why cloud providers can sell "guaranteed vCPU shares" to different customers on the same physical hardware** — this is a direct, practical application of proportional-share scheduling (Topic 3): explicit ticket-like allocations translate into guaranteed relative CPU percentages.
- **Why pinning a performance-critical process to a specific CPU core ("CPU affinity" settings in many OSes) can measurably improve its performance** — this is cache affinity (Topic 4) made an explicit, user-controllable setting rather than left entirely to the scheduler's own heuristics.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| MLFQ vs. lottery scheduling | MLFQ infers likely process behavior from history to approximate good turnaround/response time. Lottery scheduling explicitly guarantees a chosen relative CPU share via tickets and randomness — a fundamentally different goal, not a competing implementation of the same one. |
| Lottery scheduling vs. stride scheduling | Both implement proportional share. Lottery uses randomness (correct only on long-run average, simple, stateless). Stride is deterministic (correct even over short intervals, but requires tracking each process's running pass value). |
| Single-queue vs. multi-queue multiprocessor scheduling | Single-queue naturally balances load but suffers lock contention and weaker cache affinity. Multi-queue avoids lock contention and preserves cache affinity better, but needs explicit load balancing to avoid per-core imbalance. |

## What's Next

Modules 02–05 completed the first half of Virtualization: the OS's handling of the CPU — processes, the mechanism that runs them safely, and the policies that decide who runs when. **Module 06 — Memory API and Address Spaces** begins Virtualization's second half: memory. It starts at the application level (the malloc/free family and the memory bugs that come with manual memory management) before introducing the address space abstraction that Modules 07–11 will spend the rest of Virtualization building the actual translation and management machinery for.
