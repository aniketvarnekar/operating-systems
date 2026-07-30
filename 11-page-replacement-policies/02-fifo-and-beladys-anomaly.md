# FIFO and Belady's Anomaly

## Learning Objectives

By the end of this section you should be able to:
- Describe how FIFO page replacement works
- Explain Belady's Anomaly with a concrete example
- Explain why Belady's Anomaly is considered a genuinely surprising, cautionary result

## Prerequisites

- Topic 1 (The Optimal Replacement Policy)
- Module 04, Topic 2 (First-In, First-Out Scheduling) — the same name, a different context

## Motivation

Topic 1 established the theoretical best-possible policy — unimplementable, but a useful yardstick. This topic covers the simplest *implementable* policy, directly reusing an idea you've already seen in a different context (Module 04's FIFO scheduling), and shows a specific, famous result about it that should make you cautious about trusting "simple" to also mean "well-behaved."

## Problem Statement

Since Belady's optimal policy (Topic 1) requires unavailable future knowledge, what is the simplest possible practical alternative — one requiring no prediction at all, just straightforward bookkeeping? And once you have such a simple policy, is it safe to assume that giving it *more* physical memory to work with can only ever help, never hurt?

## Concept

### FIFO Page Replacement

> **FIFO (First-In, First-Out) page replacement** evicts whichever currently-resident page has been in physical memory the **longest**, regardless of how recently or frequently it's actually been accessed — tracking pages in the order they arrived, and evicting from the front of that queue when a replacement is needed.

This is conceptually identical in spirit to FIFO CPU scheduling (Module 04, Topic 2) — simple, requires no information beyond arrival order, and (as with FIFO scheduling's convoy effect) this simplicity comes at a real cost: a page that has been resident a long time but is still being actively, frequently used gets evicted purely because of its age, with no regard for whether it's actually still needed.

### Belady's Anomaly

Here is the specific, famous, genuinely counterintuitive result associated with FIFO:

> **Belady's Anomaly** is the surprising phenomenon where, under FIFO page replacement, **increasing the amount of available physical memory can actually increase the total number of page faults**, for certain specific access patterns — the exact opposite of what intuition would predict.

Intuitively, you'd expect that having *more* physical memory available should never make things worse — at worst, it should have no effect, and normally it should reduce faults by letting more pages stay resident at once. Belady's Anomaly shows this intuition is simply **false** for FIFO specifically: there exist real access sequences where FIFO produces *more* total page faults with, say, 4 available frames than it does with only 3.

This result is significant precisely because it violates a piece of intuition that seems like it should be a safe, general law of how memory systems behave — and the fact that it doesn't hold, at least for FIFO, is a genuine, well-documented cautionary result in OS history, often used to illustrate that a policy's behavior must be verified rigorously rather than assumed from "obvious" intuition.

### Why FIFO Is Susceptible to This

FIFO's eviction decision is based purely on a page's arrival time, completely disconnected from whether that page is actually still being used. Adding more available frames changes the entire *timing* of which pages happen to still be resident (and in what order) when future accesses occur — since FIFO's decisions don't track actual usage at all, this timing shift can, for specific access patterns, coincidentally cause more pages to need to be re-fetched later, rather than fewer. A policy that instead tracks actual usage (Topic 3, LRU) does not suffer from this anomaly, precisely because its eviction choices are tied to genuine access behavior rather than arbitrary arrival order.

## Internal Working (Preview)

```
 FIFO's core rule:

   maintain a queue of resident pages, in arrival order
   on a page fault, if RAM is full:
       evict the page at the FRONT of the queue (oldest arrival)
       add the newly-faulted-in page to the BACK of the queue

 Belady's Anomaly (illustrative, not a full worked trace):

   With 3 frames available: FIFO produces, say, 9 total page faults
   With 4 frames available: FIFO produces, say, 10 total page faults
                             (MORE faults, despite MORE memory!)

   This specific reversal is only possible because FIFO's eviction
   choice depends purely on arrival order, not on actual usage —
   adding a frame shifts WHICH pages are resident at each future
   access in a way that, for certain sequences, works out worse.
```

## Real-World Analogy

Think of a small pantry that always throws out whichever ingredient has been sitting on the shelf the longest, regardless of whether you still use that ingredient all the time. Common sense says a bigger pantry should only ever help — you can keep more things on hand, so you should run out of things less often. Belady's Anomaly is like discovering that, for some very particular, oddly-specific shopping and cooking pattern, a slightly bigger pantry with this exact "throw out the oldest thing" rule can actually cause you to run out of specific ingredients *more* often than the smaller pantry did — not because bigger pantries are generally bad, but because this particular "oldest first" rule, combined with this particular sequence of needs, produces a genuinely non-obvious, counterintuitive result.

## Why This Result Matters

Belady's Anomaly is a powerful, concrete demonstration that a policy's *simplicity* (FIFO requires almost no bookkeeping beyond arrival order) says nothing about whether its behavior is *intuitive* or *well-behaved* across all scenarios. It's a specific historical reason FIFO, despite its simplicity, is rarely used as the primary page-replacement policy in serious, real-world systems — practical systems instead favor policies (Topic 3) whose eviction decisions are tied to actual usage patterns, which do not exhibit this specific anomaly.

## Advantages of FIFO

- **Extremely simple to implement** — just a queue, no per-access bookkeeping needed beyond recording arrival order.
- **Low overhead** — no need to track or update usage information on every single memory access.

## Disadvantages of FIFO

- **Ignores actual usage entirely** — a frequently-accessed page can be evicted purely because it happened to arrive first, exactly analogous to how FIFO scheduling (Module 04, Topic 2) ignores job length entirely.
- **Susceptible to Belady's Anomaly** — a genuinely surprising, undesirable property where more physical memory can, for specific access patterns, produce more page faults rather than fewer.

## Best Practices

- Never assume "more memory can only help" is a universal law when evaluating a specific replacement policy — Belady's Anomaly is the concrete, well-documented counterexample proving this assumption can fail, at least for FIFO.
- When choosing (or explaining the choice of) a replacement policy for a real system, treat FIFO's simplicity as a real cost-benefit trade-off, not a free win — its ignorance of actual usage patterns is a genuine practical weakness, independent of the anomaly itself.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Adding more physical memory can never make page-fault behavior worse, for any replacement policy." | Belady's Anomaly is a specific, real, documented counterexample for FIFO: certain access patterns produce more total page faults with more available frames, not fewer — this intuition, while true for some policies, is not universally true. |
| "FIFO page replacement and FIFO CPU scheduling are unrelated ideas that just happen to share a name." | They share the exact same core principle — order strictly by arrival time, ignore any other information (job length for scheduling, actual usage for replacement) — applied to two different resources (CPU time vs. physical memory frames). |

## Interview Questions

1. **Q: How does FIFO page replacement decide which page to evict?**
   A: It evicts whichever currently-resident page has been in physical memory the longest, based purely on arrival order — with no regard for how recently or frequently that page has actually been accessed.

2. **Q: What is Belady's Anomaly?**
   A: The counterintuitive phenomenon where, under FIFO page replacement, increasing the amount of available physical memory can actually increase the total number of page faults for certain specific access patterns — the opposite of the usual expectation that more memory should only help.

3. **Q: Why is FIFO specifically susceptible to Belady's Anomaly, while a usage-aware policy like LRU is not?**
   A: FIFO's eviction decisions depend purely on arrival order, completely disconnected from actual access patterns; adding more frames shifts which pages are resident at each future access in a way that can coincidentally produce worse outcomes for certain sequences. A policy that bases eviction on actual usage (like LRU, Topic 3) ties its decisions to genuine access behavior, which does not exhibit this specific anomaly.

## Summary

- FIFO page replacement evicts the longest-resident page, based purely on arrival order, ignoring actual usage — directly analogous to FIFO CPU scheduling's ignorance of job length.
- Belady's Anomaly shows that, for certain access patterns under FIFO, adding more physical memory can actually increase total page faults — a genuinely counterintuitive, well-documented result.
- This is a specific, concrete reason FIFO is rarely used as a primary replacement policy in serious real-world systems.
- The next topic covers LRU, a usage-aware policy that avoids this anomaly, and the Clock algorithm real systems use to approximate it cheaply.
