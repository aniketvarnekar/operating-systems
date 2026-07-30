# Module 11 — Page Replacement Policies

## Module Goal

By the end of this module, you will understand **how the OS decides which resident page to evict when physical memory is full and a new page needs to come in** — the policy question Module 10 deliberately deferred at every turn. You'll learn the theoretical best-possible policy (used purely as a yardstick), the classic practical algorithms that approximate it, and thrashing: what happens when no policy can save a system from having genuinely too little physical memory for its workload.

## Topics Covered in This Module

1. **[The Optimal Replacement Policy](01-the-optimal-replacement-policy.md)** — Belady's algorithm: evict whichever page will be used furthest in the future — provably the best possible choice, and why it's unimplementable in practice.
2. **[FIFO and Belady's Anomaly](02-fifo-and-beladys-anomaly.md)** — The simplest practical policy, and the deeply counterintuitive failure mode that shows simplicity alone doesn't guarantee sane behavior.
3. **[LRU and Practical Approximations](03-lru-and-practical-approximations.md)** — Least-Recently-Used as a practical stand-in for the future, and the Clock algorithm real systems use to approximate it cheaply.
4. **[Thrashing](04-thrashing.md)** — What happens when a system's total memory demand so far exceeds physical RAM that no replacement policy can help.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 10 in full — this module directly answers the "which page to evict" question deferred throughout Module 10's page-fault handling path.

## How to Study This Module

Read in order. Topic 1 gives you a yardstick — the provably best possible policy — even though it can't actually be built, because every later policy in this module is explicitly judged against it. Topic 2 shows that "simple" and "sane" aren't the same thing, using one of the more famous counterintuitive results in OS history. Topic 3 shows the practical policy real systems actually converge on, and the clever hardware-friendly trick (the Clock algorithm) used to approximate it cheaply. Topic 4 is a necessary reality check: no replacement policy, however good, can rescue a system that simply doesn't have enough physical memory for its workload — recognizing this failure mode (thrashing) is as important as knowing the policies themselves.
