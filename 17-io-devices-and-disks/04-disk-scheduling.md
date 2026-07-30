# Disk Scheduling

## Learning Objectives

By the end of this section you should be able to:
- Explain the disk scheduling problem and its direct parallel to CPU scheduling
- Describe Shortest Seek Time First (SSTF) and its starvation risk
- Describe SCAN and C-LOOK, and how they fix SSTF's fairness problem

## Prerequisites

- Topic 3 (Hard Disk Drive Mechanics)
- Module 04 in full — the mechanism/policy split and fairness-vs-performance tension this topic directly parallels

## Motivation

Topic 3 established that seek time dominates disk access cost, and that request *order* can meaningfully affect total seek distance. This topic covers the classic policies for choosing that order — a direct, deliberate echo of Module 04's CPU scheduling, now applied to disk requests instead of CPU jobs.

## Problem Statement

Suppose several pending disk read/write requests, for sectors on different tracks, have all arrived at roughly the same time — exactly the situation Module 04, Topic 1 examined for CPU jobs, but now for physical disk locations. If the disk arm services these requests in the arbitrary order they happened to arrive (a disk equivalent of FIFO, Module 04, Topic 2), it might make several long, expensive, back-and-forth seeks across the platter that a smarter ordering could have avoided. Is there a better way to choose which pending request to service next?

## Concept

### Shortest Seek Time First (SSTF)

> **Shortest Seek Time First (SSTF)** services whichever pending request is physically **closest** to the disk arm's current position next, minimizing the immediate seek distance for each individual choice — directly analogous to Shortest Job First (Module 04, Topic 3), but minimizing seek distance instead of job length.

This significantly reduces total arm movement compared to servicing requests in arbitrary arrival order, exactly as SJF reduced average turnaround time compared to FIFO (Module 04, Topics 2–3).

### SSTF's Starvation Risk

SSTF has a direct, familiar weakness: if a steady stream of new requests keeps arriving physically close to the disk arm's current position, a request for a **physically distant** track can be repeatedly passed over in favor of closer ones, waiting indefinitely — a disk-specific instance of the same starvation risk Module 05, Topic 2 covered for MLFQ, and conceptually related to how SJF/STCF (Module 04, Topics 3–4) can leave a longer job waiting a long time under sustained pressure from shorter ones.

### SCAN: The Elevator Algorithm

> **SCAN** (often called the "elevator algorithm") moves the disk arm consistently in **one direction** (say, from the innermost track outward), servicing every pending request it passes along the way, until it reaches the outer edge — then reverses direction and sweeps back the other way, again servicing every request it passes, repeating indefinitely.

This directly fixes SSTF's starvation problem: because the arm sweeps the *entire* disk in each pass, **every** pending request is guaranteed to be serviced within, at most, one full sweep — no request, however physically distant from the current cluster of activity, can be starved indefinitely, since the arm is guaranteed to eventually pass its location.

### C-LOOK: A Refinement of SCAN

> **C-LOOK** (a common practical refinement of SCAN) behaves similarly, but instead of sweeping all the way to the disk's physical edge every time (even if there are no pending requests out there), it only travels as far as the **furthest pending request** in the current direction, then jumps back to service the closest pending request in the other direction — rather than continuing to travel (and then sweep back across) empty, request-free space.

This avoids SCAN's minor inefficiency of sometimes traveling all the way to the physical edge of the disk even when no actual requests are waiting out there, while preserving SCAN's core fairness guarantee (every request gets serviced within a bounded sweep).

## Internal Working (Preview)

```
   SSTF: always service the CLOSEST pending request next

     Arm at track 50, pending requests at: 52, 48, 500
     → services 52 (closest), then 48 (closest remaining)...
       500 waits and waits if MORE close requests keep arriving —
       STARVATION RISK


   SCAN: sweep in ONE direction, servicing everything passed,
         then reverse

     Arm sweeps: 0 → 50 → 52 → 300 → 500 → ... → END of disk
                 (services every request encountered along the way)
     then reverses: END → ... → back toward 0
     → EVERY request guaranteed service within one full sweep


   C-LOOK: like SCAN, but only travel as far as the LAST pending
           request, then jump back — skip the empty tail

     Arm sweeps: 0 → 50 → 52 → 300 → 500 (last pending request —
                 STOP here, don't continue to the physical disk edge)
     then jumps directly back to the closest pending request
     in the other direction
```

## Real-World Analogy

Recall this module's earlier vinyl-record-library analogy (Topic 3), or think of a building elevator instead. SSTF is like an elevator that always goes to whichever floor currently has someone waiting closest to its current position — efficient in the short term, but someone waiting on a floor far from the current cluster of activity could wait a very long time if closer requests keep coming in. SCAN is like an elevator with a fixed policy: always continue in the current direction (up or down), stopping at every floor with a waiting passenger along the way, only reversing direction once it reaches the very top or bottom — guaranteeing everyone gets picked up within, at worst, one full up-and-down cycle. C-LOOK is the same elevator, but smart enough not to keep going all the way to the physical top floor if nobody up there is actually waiting — it only goes as far as the highest floor with an actual waiting passenger, then reverses.

## Why This Design Mirrors CPU Scheduling So Closely

Disk scheduling and CPU scheduling (Module 04) are solving structurally the same problem: given several waiting requests and a scarce, shared resource (the disk arm's position; the CPU), choose an order that balances raw efficiency (minimizing total seek distance; minimizing average turnaround time) against fairness (no request/job starved indefinitely). SSTF's starvation mirrors SJF/STCF's fairness cost; SCAN/C-LOOK's guaranteed-bounded-wait property mirrors Round Robin's guaranteed-turn property (Module 04, Topic 5) — the same underlying tension between pure efficiency and fairness reappears here, just applied to a physically different resource.

## Advantages of SCAN/C-LOOK Over SSTF

- **Bounded worst-case wait** — every request is guaranteed service within one sweep (or a bounded number of sweeps), eliminating SSTF's starvation risk.
- **Still substantially better than purely arbitrary (FIFO-style) ordering** — the sweeping pattern naturally batches nearby requests together, capturing most of SSTF's efficiency benefit without its fairness cost.

## Disadvantages / Trade-offs

- **Not quite as locally optimal as SSTF** for any single, immediate decision — SCAN/C-LOOK may occasionally pass up a very close request in favor of continuing its current sweep direction, in exchange for the fairness guarantee.
- **The specific choice of scheduling policy is one factor among several** (Topic 3's broader mechanical realities, and RAID's parallelism, Module 18) in overall disk subsystem performance — no scheduling policy alone eliminates the fundamental seek/rotation costs from Topic 3.

## Best Practices

- When comparing disk scheduling policies, explicitly frame the discussion around the same efficiency-vs-fairness tension used for CPU scheduling (Module 04) — the parallel is deliberate and makes the trade-offs easy to reason about using already-familiar vocabulary.
- Recognize C-LOOK (or an equivalent practical refinement) as the more realistic real-world choice over textbook-pure SCAN, since avoiding unnecessary travel to an empty disk edge is a straightforward, low-cost improvement.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "SSTF is always the best disk scheduling policy since it minimizes seek time for every decision." | SSTF can starve requests for physically distant tracks if closer requests keep arriving — it optimizes each individual decision locally but offers no fairness guarantee, exactly like SJF/STCF's fairness cost for CPU scheduling (Module 04). |
| "SCAN and C-LOOK are essentially the same algorithm with no meaningful difference." | SCAN always sweeps all the way to the disk's physical edge; C-LOOK only travels as far as the furthest actual pending request before reversing — a real, practical efficiency difference when requests don't span the entire disk. |

## Interview Questions

1. **Q: What is SSTF, and what is its main weakness?**
   A: Shortest Seek Time First services whichever pending request is physically closest to the disk arm's current position next, minimizing immediate seek distance. Its weakness is starvation risk: a request for a physically distant track can be repeatedly passed over if closer requests keep arriving.

2. **Q: How does SCAN fix SSTF's starvation problem?**
   A: SCAN sweeps the disk arm consistently in one direction, servicing every request it passes, then reverses — guaranteeing every pending request is serviced within at most one full sweep, regardless of how physically distant it is from the current cluster of activity.

3. **Q: How does C-LOOK improve on SCAN?**
   A: Instead of always sweeping all the way to the disk's physical edge, C-LOOK only travels as far as the furthest actual pending request in the current direction before reversing, avoiding unnecessary travel across empty, request-free space while preserving SCAN's fairness guarantee.

## Summary

- Disk scheduling chooses the order to service pending disk requests, directly paralleling CPU scheduling's efficiency-vs-fairness tension (Module 04).
- SSTF services the closest pending request first, minimizing immediate seek distance but risking starvation for physically distant requests.
- SCAN sweeps the disk arm in one direction, servicing everything passed, then reverses — guaranteeing bounded wait for every request; C-LOOK refines this by not sweeping past the last actual pending request.
- This closes out the module's coverage of I/O devices and disks — the module summary ties the canonical device protocol, drivers, disk mechanics, and scheduling together before Module 18 covers RAID, which combines multiple disks for capacity, performance, or reliability.
