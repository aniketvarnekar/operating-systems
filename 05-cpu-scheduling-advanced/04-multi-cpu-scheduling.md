# Multi-CPU Scheduling

## Learning Objectives

By the end of this section you should be able to:
- Explain what cache affinity is and why it matters for multi-CPU scheduling
- Compare single-queue and multi-queue multiprocessor scheduling designs
- Explain what load balancing is and why it's needed even with a per-core-queue design

## Prerequisites

- Module 04 in full
- Topic 1 (The Multi-Level Feedback Queue) — helpful context, not required

## Motivation

Every scheduling policy covered so far — in this module and Module 04 — quietly assumed there is exactly **one** CPU making scheduling decisions. Nearly every real computer today has multiple CPU cores. This topic covers what genuinely changes (not just "run the same algorithm on each core independently") once multiple cores are scheduling from a shared pool of ready processes.

## Problem Statement

With multiple CPU cores, a new question appears that simply didn't exist with one core: if there are, say, four ready processes and four available cores, do you run all four processes on whichever core happens to be free at the moment (potentially moving a given process to a *different* core than it ran on last time)? Or do you try to keep each process consistently running on the *same* core it ran on before? And if processes are grouped into per-core ready queues rather than one shared queue, what happens if one core's queue becomes empty while another core's queue is overloaded with several waiting processes?

## Concept

### Cache Affinity

Modern CPUs rely heavily on small, extremely fast memory caches physically located on (or very near) each core, which retain a copy of data and instructions a process has recently used — dramatically speeding up subsequent accesses to that same data, as long as it's still present in the cache. This is directly relevant to scheduling:

> **Cache affinity** refers to the performance advantage a process gains from continuing to run on the *same* CPU core it ran on previously, since that core's cache may still hold data the process will access again — avoiding the cost of re-fetching that data from slower main memory.

If a scheduler moves a process to a *different* core every time it runs, that new core's cache starts "cold" for this process — none of its recently-used data is present there yet, forcing potentially expensive re-fetches from main memory that a same-core resumption would have avoided. A cache-affinity-aware scheduler deliberately tries to keep a given process on the same core across successive runs whenever reasonably possible, specifically to preserve this performance advantage.

### Single-Queue vs. Multi-Queue Scheduling

Two broad architectural approaches exist for organizing ready processes across multiple cores:

> **Single-Queue Multiprocessor Scheduling (SQMS)** keeps one shared queue of all ready processes, with any idle core pulling the next process from that single shared queue.

- **Advantage**: naturally balances load, since any free core simply takes the next available process — no core sits idle while another has a backlog.
- **Disadvantage**: the single shared queue must be protected by locking (a concurrency mechanism previewed here and covered fully in Module 13) to prevent multiple cores from grabbing the same process simultaneously, and this shared-lock contention becomes a real bottleneck as the number of cores grows. It also makes preserving cache affinity harder, since any process could end up on any core each time it's scheduled.

> **Multi-Queue Multiprocessor Scheduling (MQMS)** gives each core its own separate, independent ready queue, with each core scheduling only from its own queue.

- **Advantage**: no shared-queue lock contention (each core's queue is independent), and it's naturally easier to keep a given process consistently on the same core (good cache affinity), since a process simply stays in that core's own queue between runs.
- **Disadvantage**: can suffer from **load imbalance** — one core's queue might have several waiting processes while another core's queue sits empty, wasting that idle core's capacity even though there's ready work elsewhere in the system.

### Load Balancing

> **Load balancing** is the technique of periodically moving (migrating) a process from an overloaded core's queue to an idle or lightly-loaded core's queue, specifically to address MQMS's load-imbalance weakness.

This directly creates a trade-off with cache affinity: migrating a process to a different core to fix load imbalance means sacrificing whatever cache affinity it had built up on its original core. A well-designed multi-queue scheduler must balance both concerns — avoid migrating processes so often that cache affinity is constantly and needlessly sacrificed, but migrate often enough that no core sits idle while meaningful backlog exists elsewhere.

## Internal Working (Preview)

```
 Single-Queue Multiprocessor Scheduling (SQMS):

   ┌─────────────────────────┐
   │   One shared ready queue  │
   └────────────┬─────────────┘
        ┌────────┼────────┬────────┐
        ▼        ▼        ▼        ▼
     Core 1   Core 2   Core 3   Core 4
   (any idle core pulls the next process — naturally balanced,
    but the shared queue needs locking, and cache affinity suffers)


 Multi-Queue Multiprocessor Scheduling (MQMS):

     Core 1        Core 2        Core 3        Core 4
   ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
   │ Queue 1 │    │ Queue 2 │    │ Queue 3 │    │ Queue 4 │
   │ [P, Q]  │    │ [ ]     │    │ [R]     │    │ [S, T,U]│
   └────────┘    └────────┘    └────────┘    └────────┘
   (no shared-lock contention, good cache affinity — but Core 2
    sits idle while Core 4 has a backlog: LOAD IMBALANCE)
                          │
                          ▼
              periodic LOAD BALANCING migrates
              a process from an overloaded queue
              to an idle/lighter one
```

## Real-World Analogy

Think of a large restaurant with four separate kitchen stations (cores), each with its own dedicated line cook. **Single-queue** is like having one shared order line that any free cook grabs the next ticket from — naturally balanced (no cook stands idle while tickets pile up elsewhere), but cooks constantly switching to unfamiliar dishes lose the benefit of already having their specific station's ingredients prepped and within reach (cache affinity), and everyone reaching for the same shared ticket rail creates congestion (lock contention). **Multi-queue** is like giving each cook their own dedicated ticket rail for orders assigned specifically to them — each cook stays efficient with their own prepped station (good cache affinity), and there's no congestion over a shared rail, but one cook's rail might pile up with five tickets while another cook's rail sits completely empty (load imbalance) unless a manager periodically walks over and manually reassigns a ticket from the overloaded rail to the idle cook (load balancing) — at the cost of that reassigned order losing whatever head-start prep the original cook's station had already done for it.

## Why This Design Space Involves Real Trade-offs

Multi-CPU scheduling cannot simply optimize for one goal in isolation: perfect load balance (achieved trivially by SQMS's single shared queue) directly conflicts with minimizing lock contention and preserving cache affinity (which MQMS's independent per-core queues achieve, but only by risking load imbalance). Real-world general-purpose OS schedulers (like Linux's) generally adopt an MQMS-style design specifically for its lock-contention and cache-affinity advantages, and then add deliberate, tunable load-balancing logic on top to control the second weakness directly, rather than accepting either extreme.

## Advantages of Multi-Queue Designs (With Load Balancing)

- **Better scalability** — avoiding a single shared, heavily-contended queue lock as the number of cores grows.
- **Preserves cache affinity** for the common case, while load balancing specifically handles the exceptional case of genuine imbalance, rather than sacrificing affinity on every single scheduling decision.

## Disadvantages / Trade-offs

- **Tuning complexity** — deciding how aggressively (and how often) to migrate processes for load balancing directly trades off against how much cache affinity benefit is preserved; there's no universally correct setting, only a workload-dependent balance.
- **Migration itself has a real cost** — moving a process to a different core's queue isn't free; it discards that process's accumulated cache-affinity benefit on its prior core.

## Best Practices

- When reasoning about multi-core scheduler behavior, always separately ask: "is load balanced across cores?" and "is cache affinity being preserved?" — treating these as one combined concern obscures the real trade-off between them.
- Recognize that real-world schedulers (Linux's CFS and similar) are essentially MQMS with tuned load balancing — not pure SQMS — precisely because lock contention and cache-affinity loss are serious costs at realistic core counts.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Running the exact same single-core scheduling algorithm independently on each core is sufficient for a multi-core system." | It ignores cache affinity (whether a process consistently returns to the same core) and load balancing (whether work is evenly distributed across cores) — both are genuinely new concerns that don't exist at all in a single-CPU model. |
| "A single shared ready queue across all cores has no real downside compared to per-core queues." | It requires locking to prevent multiple cores from grabbing the same process simultaneously, and that shared lock becomes a real point of contention as core count grows — plus it makes preserving cache affinity for any given process significantly harder. |

## Interview Questions

1. **Q: What is cache affinity, and why does it matter for multi-CPU scheduling?**
   A: The performance benefit a process gets from running on the same CPU core it ran on before, since that core's cache may still hold data the process will reuse. Moving a process to a different core forces that core's cache to "warm up" from scratch, incurring slower main-memory accesses it could have otherwise avoided.

2. **Q: What's the fundamental trade-off between single-queue and multi-queue multiprocessor scheduling?**
   A: Single-queue (SQMS) naturally balances load across cores but requires shared-queue locking (a scalability bottleneck) and makes cache affinity harder to preserve. Multi-queue (MQMS) avoids shared-lock contention and preserves cache affinity better, but can suffer load imbalance between cores' independent queues.

3. **Q: Why is load balancing needed even in a well-designed multi-queue scheduler?**
   A: Because independent per-core queues can naturally drift out of balance over time — one core's queue may accumulate several waiting processes while another's sits empty — wasting the idle core's capacity unless processes are periodically migrated from overloaded queues to lighter ones.

## Summary

- Multi-CPU scheduling introduces genuinely new concerns beyond Module 04's single-CPU model: cache affinity (staying on the same core for cache reuse) and load balancing (keeping work evenly distributed across cores).
- Single-queue designs balance load naturally but suffer from lock contention and weaker cache affinity; multi-queue designs avoid these but risk load imbalance.
- Load balancing directly trades off against cache affinity — migrating a process to fix imbalance discards its accumulated cache benefit on the original core.
- This closes out the module's advanced scheduling topics — the module summary ties MLFQ, lottery/proportional-share, and multi-CPU scheduling together, and completes Virtualization's CPU-focused half of this course before Module 06 begins the memory half.
