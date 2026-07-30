# LRU and Practical Approximations

## Learning Objectives

By the end of this section you should be able to:
- Explain the principle of locality that motivates LRU, connecting it back to Module 09's TLB discussion
- Explain why exact LRU is expensive to implement precisely
- Describe the Clock algorithm and explain how it approximates LRU cheaply using a reference bit

## Prerequisites

- Topic 1 (The Optimal Replacement Policy)
- Topic 2 (FIFO and Belady's Anomaly)
- Module 09, Topic 1 (The Translation Lookaside Buffer) — specifically, locality of reference

## Motivation

Topic 1 gave you the unimplementable ideal; Topic 2 gave you the simplest implementable policy, along with a specific reason to distrust it. This topic covers the policy real-world systems actually converge on in spirit — one that uses the past as a practical stand-in for the unknowable future — and the clever, cheap trick used to approximate it without excessive bookkeeping overhead.

## Problem Statement

Belady's optimal policy (Topic 1) evicts the page needed furthest in the *future* — information no real system has. But recall locality of reference from Module 09, Topic 1: real programs tend to re-access the same pages repeatedly within a short window of time. If the future is unknowable, could the **recent past** serve as a reasonable, practical stand-in for predicting it?

## Concept

### Least Recently Used (LRU)

> **LRU (Least Recently Used)** page replacement evicts whichever currently-resident page has gone the **longest without being accessed** — using recency of past use as a practical proxy for likelihood of future use, directly leveraging the same locality-of-reference principle that makes the TLB effective (Module 09, Topic 1).

The intuition: a page that hasn't been touched in a long time is, empirically, less likely to be needed again soon than a page that was just accessed a moment ago — the mirror image of Belady's optimal rule (evict what's needed furthest in the *future*), but looking backward at what's already observable (the past) instead of forward at what isn't.

### Why Exact LRU Is Expensive

To implement LRU *exactly*, the OS would need to know, with precision, the exact relative order in which every single resident page was most recently accessed — updated on **every single memory reference**, not just on page faults. This requires either maintaining a full, continuously-updated ordering (a timestamp or a linked-list position updated on every access) or equivalent bookkeeping — a genuinely significant overhead if applied to every single memory reference a process makes, directly undermining the efficiency goal (Module 06, Topic 1) that motivated the TLB (Module 09, Topic 1) in the first place. Precise LRU tracking at this granularity is generally considered too expensive for practical, general-purpose use.

### The Clock Algorithm: A Cheap Approximation

> The **Clock algorithm** approximates LRU using a single **reference bit** per page (set by hardware automatically whenever that page is accessed) and a circular scan through resident pages, rather than tracking a full, precise recency ordering.

The mechanism:
1. Resident pages are conceptually arranged in a circle, with a single "clock hand" pointing to one of them.
2. When a replacement is needed, the algorithm examines the page currently pointed to by the hand:
   - If its reference bit is **0** (not recently accessed), evict it.
   - If its reference bit is **1** (recently accessed), **clear it to 0** (giving it a "second chance") and advance the hand to the next page, repeating this check.
3. The hand keeps advancing around the circle until it finds a page with a reference bit of 0 to evict.

This gives pages that have been accessed recently a genuine "second chance" (their bit gets cleared, but they're not evicted immediately) before ultimately being evicted if they're never accessed again by the time the hand comes back around to them — approximating "evict something not recently used" without the overhead of tracking a fully precise, continuously-updated recency order for every page.

## Internal Working (Preview)

```
   Clock algorithm — pages arranged in a circle, each with a reference bit:

        ┌───┐     ┌───┐     ┌───┐     ┌───┐
        │ A   │───►│ B   │───►│ C   │───►│ D   │──┐
        │ ref=1│     │ ref=0│     │ ref=1│     │ ref=0│  │
        └───┘     └───┘     └───┘     └───┘  │
          ▲                                          │
          └──────────────────────────────────────────┘
                    (circular — wraps back to A)

    Clock hand currently at A (ref=1):
      → give A a second chance: clear its bit to 0, advance hand to B

    Hand now at B (ref=0):
      → EVICT B (not recently accessed since its bit was last cleared
         or since it was loaded)
```

### Why This Approximates LRU Well Enough

Exact LRU would require knowing the *precise* order of all past accesses; the Clock algorithm settles for a coarser, cheaper distinction: "has this page been accessed at all since the last time the hand passed by" (one bit, hardware-maintained, essentially free to check) rather than "exactly when, relative to every other page, was this page last accessed" (a much more expensive, precisely-ordered record). In practice, this coarser approximation performs close enough to true LRU for the vast majority of real workloads, at a dramatically lower bookkeeping cost — a classic engineering trade-off between precision and practicality, similar in spirit to MLFQ's use of observed behavior as a proxy for exact knowledge (Module 05, Topic 1).

## Real-World Analogy

Think of exact LRU like a librarian who keeps a perfectly precise, constantly-updated list ranking every single book in the library by the exact moment it was last checked out, down to the second — accurate, but an enormous amount of continuous record-keeping for a very large collection. The Clock algorithm is like a librarian who instead just walks around the shelves in a fixed loop, and for each book, checks a simple "was this touched since my last lap" flag: if not flagged, pull it for removal; if flagged, just reset the flag (giving it one more lap's grace period) and keep walking. This gets you almost the same practical outcome — books nobody's touched in a while eventually get removed — without ever needing the librarian to maintain a perfectly precise, constantly-updated ranking of every single book.

## Why This Design Is Necessary

Exact LRU's overhead (updating a precise ordering on every single memory access) would be a serious efficiency cost, directly undermining Module 06, Topic 1's efficiency goal — precisely the same kind of concern that motivated the TLB (Module 09, Topic 1) in the first place. The Clock algorithm resolves this tension by using a much cheaper signal (a single hardware-maintained reference bit per page, checked only during the comparatively rare event of an actual replacement decision) instead of continuous, precise bookkeeping on every access — trading a small amount of replacement-quality precision for a dramatic reduction in overhead.

## Advantages of LRU / Clock

- **Grounded in real, empirically-validated locality of reference**, rather than an arbitrary rule like FIFO's pure arrival order — this makes LRU-style policies generally much more effective in practice.
- **Does not exhibit Belady's Anomaly** — because eviction decisions are tied to actual usage patterns rather than arbitrary arrival timing, LRU avoids the specific counterintuitive failure mode from Topic 2.
- **The Clock algorithm achieves this with minimal overhead** — a single hardware-maintained bit per page, checked only during replacement decisions, rather than continuous, precise bookkeeping on every single memory access.

## Disadvantages / Limitations

- **Exact LRU is too expensive for practical, general-purpose use** — precise, continuously-updated recency tracking on every memory access is a real, serious overhead.
- **The Clock algorithm is only an approximation** — it can occasionally make a suboptimal eviction choice that exact LRU (or the optimal policy, Topic 1) would not have made, since it only distinguishes "accessed since last sweep" from "not," rather than a fully precise recency ordering.
- **Like all practical policies, it can still perform poorly** if a workload's actual access pattern doesn't exhibit strong locality — no history-based approximation can help if the past genuinely isn't predictive of the near future for a given workload.

## Best Practices

- When explaining why LRU-style policies are preferred over FIFO in real systems, lead with locality of reference as the underlying justification — it's the same empirical property that made the TLB (Module 09, Topic 1) effective, applied here to a different problem.
- Recognize the Clock algorithm as a recurring engineering pattern: approximate an expensive-but-ideal signal (precise recency) with a cheap-but-good-enough one (a single reference bit, periodically reset) — a pattern worth recognizing across many areas of systems design, not just this one.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "LRU and the Clock algorithm are the same thing." | LRU is the ideal policy (evict the page least recently used, precisely). The Clock algorithm is a practical, cheaper approximation of that ideal, using a single reference bit and a circular scan rather than a fully precise recency ordering — it can occasionally diverge from what exact LRU would choose. |
| "Exact LRU is used in most real operating systems because it's clearly the best practical policy." | Exact LRU's overhead (precise, continuously-updated bookkeeping on every single memory access) is generally considered too expensive for practical, general-purpose use; real systems favor cheaper approximations like the Clock algorithm instead. |

## Interview Questions

1. **Q: What principle does LRU page replacement rely on, and why is it a reasonable substitute for the unimplementable optimal policy?**
   A: Locality of reference — the empirical observation that recently-used pages are likely to be used again soon. Since the true future (which optimal replacement, Topic 1, would need) is unknowable, LRU uses the observable, recent past as a practical, reasonably reliable stand-in for predicting near-future access.

2. **Q: Why is exact LRU considered too expensive to implement in most real systems?**
   A: It requires maintaining a precise, continuously-updated ordering of every resident page's most recent access, updated on every single memory reference — a significant bookkeeping overhead that would undermine the efficiency goal of memory virtualization.

3. **Q: How does the Clock algorithm approximate LRU cheaply?**
   A: It maintains a single hardware-set reference bit per page and scans resident pages in a circular order. A page with its bit set gets a "second chance" (the bit is cleared, and the scan moves on); a page whose bit is already 0 when the scan reaches it is evicted — approximating "not recently used" without tracking a fully precise recency order.

## Summary

- LRU evicts the page that's gone the longest without being accessed, using locality of reference (the same principle behind the TLB's effectiveness) as a practical stand-in for the unknowable future that optimal replacement would need.
- Exact LRU requires expensive, continuous bookkeeping on every memory access, making it impractical for general-purpose use.
- The Clock algorithm approximates LRU cheaply using a single hardware-maintained reference bit per page and a circular "second chance" scan, avoiding both exact LRU's overhead and FIFO's susceptibility to Belady's Anomaly.
- The next topic covers thrashing: what happens when a system's total memory demand so far exceeds physical RAM that no replacement policy, however good, can prevent severe performance collapse.
