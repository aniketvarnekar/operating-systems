# Free-Space Management

## Learning Objectives

By the end of this section you should be able to:
- Explain the general free-space management problem, independent of any one specific context
- Compare best fit, worst fit, and first fit allocation strategies, and state a real weakness of each
- Explain how splitting and coalescing free chunks help manage fragmentation over time

## Prerequisites

- Topic 2 (Segmentation) — specifically, its introduction of external fragmentation
- Module 06, Topic 2 (The Memory API) — the `malloc()`/`free()` heap this topic's ideas also directly apply to

## Motivation

Topic 2 ended with external fragmentation: variable-sized segments scattered through physical memory, leaving free space split into multiple, non-contiguous chunks. This exact problem — managing a pool of free space to satisfy variable-sized requests — isn't unique to segmentation; it's precisely the same problem a heap allocator (Module 06, Topic 2's `malloc()`/`free()`) faces internally, managing free space *within* a process's own heap. This topic covers the general problem and its classic solutions, which apply to both contexts.

## Problem Statement

Suppose a pool of free memory is fragmented into several chunks of different sizes, scattered non-contiguously (exactly the external-fragmentation scenario from Topic 2): a 10-byte free chunk here, a 30-byte free chunk there, a 100-byte free chunk elsewhere. A new request arrives for, say, 20 bytes. Which free chunk should be used to satisfy it? The choice matters — a poor choice now can make the pool harder to satisfy *future* requests efficiently, even though several chunks are large enough to work right now.

## Concept

### The Free-Space Management Problem

> **Free-space management** is the general problem of tracking a pool of free (unallocated) memory of varying chunk sizes, and deciding which specific free chunk to use when a new, variable-sized allocation request arrives — with the goal of minimizing wasted space and keeping future requests satisfiable as well.

### Best Fit

> **Best fit** searches the entire pool of free chunks and selects the smallest one that is still large enough to satisfy the request.

- **Advantage**: minimizes the amount of leftover, wasted space from any single allocation, since it picks the tightest possible fit.
- **Disadvantage**: this "tightest fit" often leaves behind a very small, awkward leftover sliver (the difference between the chosen chunk's size and the requested size) that's too small to usefully satisfy most future requests — over time, this can leave the pool full of many small, nearly-useless slivers. It also typically requires searching the *entire* list of free chunks to find the true best fit, which is slower than strategies that can stop early.

### Worst Fit

> **Worst fit** searches the entire pool and selects the **largest** available free chunk, on the theory that using the biggest chunk leaves behind the largest possible leftover remainder — a remainder more likely to still be useful for a future request, rather than an unusably small sliver.

- **Advantage**: deliberately avoids leaving behind tiny, awkward slivers.
- **Disadvantage**: consistently consumes the largest available chunks, meaning the pool tends to lose its largest chunks quickly — if a future request genuinely needs a very large contiguous chunk, none may remain, even though sufficient total free space still exists elsewhere in the pool as smaller pieces. Like best fit, it typically requires scanning the entire free list.

### First Fit

> **First fit** simply scans the pool of free chunks in order and selects the *first* one large enough to satisfy the request, without searching for the smallest or largest available option.

- **Advantage**: much faster in practice than best fit or worst fit, since it doesn't need to examine every free chunk before deciding — it stops as soon as it finds any chunk that's big enough.
- **Disadvantage**: can still leave behind awkward leftover slivers (like best fit can), and tends to leave smaller fragments clustered near the start of the free list over time, though real implementations often mitigate this by rotating the starting search point on each request (a variant called **next fit**).

### Splitting and Coalescing

Two general techniques help every one of these strategies manage fragmentation more effectively over time:

> **Splitting**: when a chosen free chunk is larger than the requested size, split it into two pieces — one exactly the size requested (handed out to satisfy the allocation), and one representing the leftover remainder (kept in the free pool for future use), rather than handing out the entire, oversized chunk and wasting the difference.

> **Coalescing**: when a chunk of memory is freed, check whether it is physically adjacent to any other already-free chunk(s), and if so, merge them back together into one single, larger free chunk.

Coalescing directly counteracts fragmentation's tendency to accumulate over time: without it, repeatedly splitting chunks to satisfy requests, then freeing them individually, would leave the pool increasingly divided into smaller and smaller pieces, permanently, even after the original allocations are no longer in use. Coalescing restores larger, contiguous free chunks whenever adjacent free space allows it.

## Internal Working (Preview)

```
 Free pool before a new 20-byte request:
   [ 10 free ] [ 30 free ] [ 100 free ]  (three separate, non-contiguous chunks)

 Best fit  → chooses the 30-byte chunk (smallest that still fits)
             → SPLIT into: 20 bytes (allocated) + 10 bytes (new, smaller free chunk)

 Worst fit → chooses the 100-byte chunk (largest available)
             → SPLIT into: 20 bytes (allocated) + 80 bytes (still-substantial free chunk)

 First fit → scans in order, picks whichever chunk it encounters FIRST
             that's ≥ 20 bytes (here, the 30-byte chunk, without
             checking whether a better option exists later in the list)


 Coalescing example:
   [ freed: 20 ] [ still allocated ] [ freed: 15 ]
   → if the middle block is later ALSO freed, and it's physically
     adjacent to both neighbors, all three merge into one 
     larger free chunk, rather than remaining three small,
     separately-tracked pieces
```

## Real-World Analogy

Think of a valet parking lot with cars of different lengths (allocation requests) and open parking spaces of different lengths (free chunks) scattered throughout the lot. **Best fit** is a valet who always looks for the smallest available space that the car still fits into, minimizing wasted space per car — but this can leave behind lots of tiny, oddly-shaped gaps too small for any other car. **Worst fit** is a valet who always parks in the largest available space, deliberately leaving a large, still-useful remainder behind — but this quickly uses up all the large spaces, leaving only medium and small ones for cars that might genuinely need more room later. **First fit** is a valet who just parks in the first space encountered that's big enough, without walking the entire lot first — much faster, but not necessarily optimal. **Coalescing** is what happens when two adjacent cars both leave at the same time — the valet recognizes the now-empty adjacent spaces can be treated as one single larger space for the next, possibly bigger car.

## Why No Single Strategy Is Universally "Best"

Each strategy makes a different bet about what future requests will look like, and none can be optimal for every possible sequence of allocations and frees — this is a well-known, fundamentally unavoidable trade-off in free-space management, not a solvable engineering gap. Real-world allocators (both OS-level segment/page managers and `malloc()`-style heap allocators, Module 06, Topic 2) typically use more sophisticated hybrid approaches (segregated free lists by size class, buddy allocation systems, and others) that blend these basic ideas to perform well across typical, realistic request patterns — but the fundamental best-fit/worst-fit/first-fit trade-offs described here remain the conceptual foundation those hybrid systems build on.

## Advantages of Understanding This Topic

- **Directly explains a real, common performance and memory-usage phenomenon**: a long-running program's memory can become progressively less efficiently packed over time (fragmentation) purely as a consequence of allocation/free patterns, independent of any actual bug.
- **Applies uniformly across contexts** — the exact same reasoning governs an OS placing segments in physical memory (Topic 2) and a `malloc()` implementation placing objects within a process's own heap (Module 06, Topic 2) — it's a single, general problem with shared solutions.

## Disadvantages / Limitations

- **No strategy avoids fragmentation entirely** — best fit, worst fit, and first fit all can degrade over time under certain allocation/free patterns; coalescing helps but cannot always fully undo fragmentation, especially if non-adjacent chunks are the ones sitting free.
- **Searching for a suitable chunk has a real cost** — best fit and worst fit's full-pool searches are slower than first fit's early-exit approach, a direct time-vs-packing-quality trade-off.

## Best Practices

- When choosing (or evaluating) a free-space management strategy, consider the actual expected pattern of allocation sizes and lifetimes for the specific workload — there's no universally correct choice independent of that context.
- Always pair a splitting strategy with a coalescing strategy — splitting without ever coalescing guarantees fragmentation only ever gets worse over time, never better.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Best fit is objectively the best strategy, since the name suggests it." | Best fit minimizes waste on a single allocation, but tends to leave many small, awkward, hard-to-reuse slivers over time — it is not universally superior to worst fit or first fit across all workloads. |
| "Coalescing can always fully eliminate fragmentation." | Coalescing only merges chunks that are freed and happen to be physically adjacent to other free chunks — if fragmented free chunks are separated by still-allocated memory, they cannot be merged no matter how much total free space exists. |

## Interview Questions

1. **Q: What's the difference between best fit, worst fit, and first fit allocation strategies?**
   A: Best fit selects the smallest free chunk still large enough for the request, minimizing per-allocation waste but risking many small, awkward leftover slivers. Worst fit selects the largest available chunk, leaving a more useful remainder but quickly depleting large chunks. First fit selects the first chunk encountered that's large enough, trading optimality for much faster search time.

2. **Q: What is splitting, and why is it necessary?**
   A: Splitting divides a chosen free chunk larger than the request into an allocated portion (exactly the requested size) and a smaller, still-free remainder — without it, satisfying a request would waste the entire size difference of an oversized chunk instead of preserving the leftover for future use.

3. **Q: What is coalescing, and what problem does it solve?**
   A: Coalescing merges a newly-freed chunk with any physically adjacent free chunks into one larger contiguous chunk. It counteracts fragmentation's tendency to accumulate over time — without it, repeated splitting and freeing would permanently divide the pool into progressively smaller, less useful pieces.

## Summary

- Free-space management is the general problem of tracking variable-sized free memory chunks and deciding which one to use for each new request — directly relevant both to segmentation's placement problem (Topic 2) and to heap allocators (Module 06, Topic 2).
- Best fit minimizes per-allocation waste but creates awkward slivers; worst fit avoids slivers but depletes large chunks quickly; first fit trades optimality for speed.
- Splitting and coalescing are general techniques that work alongside any of these strategies — splitting avoids wasting an oversized chunk's excess; coalescing merges adjacent free chunks to fight fragmentation's accumulation over time.
- This closes out the module's coverage of early address-translation mechanisms — the module summary ties base & bounds, segmentation, and free-space management together before Module 08 introduces paging, the modern technique that supersedes both.
