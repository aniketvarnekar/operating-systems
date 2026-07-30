# Module 07 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Dynamic Relocation (Base and Bounds)** — one base/bounds pair per process, hardware-enforced translation and protection, and its internal-fragmentation weakness
- [x] **Segmentation** — multiple independent base/bounds pairs per process, fixing internal fragmentation while introducing external fragmentation
- [x] **Free-Space Management** — best fit, worst fit, first fit, splitting, and coalescing, as the general solution to managing scattered free space

## The Big Picture

This module traced one continuous engineering narrative: start with the simplest possible hardware-assisted translation scheme (base & bounds), find its specific weakness (internal fragmentation), fix it with a natural generalization (segmentation), discover the new problem that generalization creates (external fragmentation), and cover the general toolkit (free-space management) for living with that problem. This same pattern — a clean mechanism, a discovered weakness, a targeted refinement, a new trade-off — is the same shape Module 05 used for MLFQ, and it reappears again as Module 08 introduces paging.

```
   Base & Bounds (Topic 1)
        │  weakness: internal fragmentation
        │  (unused heap/stack gap wastes physical memory)
        ▼
   Segmentation (Topic 2)
        │  weakness: external fragmentation
        │  (scattered free chunks, none big enough alone)
        ▼
   Free-Space Management (Topic 3)
        │  best fit / worst fit / first fit + splitting/coalescing
        │  — manages, but does not eliminate, fragmentation
        ▼
   Module 08: Paging — sidesteps fragmentation almost entirely,
   by using small, FIXED-size chunks instead of variable-sized ones
```

## Practical Connections

- **Why older or resource-constrained systems occasionally needed a "defragmentation" step, while modern virtual-memory systems mostly don't** — segmentation-style, variable-sized-chunk allocation is genuinely prone to external fragmentation (Topic 2); paging's fixed-size-chunk approach (Module 08) avoids this specific problem almost entirely, which is a major reason it became the dominant technique.
- **Why a `malloc()`-heavy C program's memory usage can look "bloated" even when the actual live data is modest** — this is external fragmentation and sliver-accumulation (Topic 3) happening inside the process's own heap, the exact same phenomenon as Topic 2's segment placement problem, just at a smaller scale.
- **Why hardware memory protection (a program crashing with a "segmentation fault") is called exactly that** — the term is a direct, historical reference to the segmentation mechanism in Topic 2: an out-of-bounds access relative to a segment's base/bounds pair is precisely what such a fault represents.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Base & bounds vs. segmentation | Base & bounds uses one base/bounds pair for the entire address space; segmentation generalizes this to multiple independent base/bounds pairs, one per logical region (Code, Heap, Stack). |
| Internal fragmentation vs. external fragmentation | Internal: wasted space *inside* one allocated/reserved block (base & bounds' unused heap/stack gap). External: wasted space *between* scattered free chunks, none individually large enough for a new request (segmentation's placement problem). |
| Best fit vs. worst fit vs. first fit | Best fit minimizes leftover waste per allocation but creates awkward slivers over time; worst fit avoids slivers but depletes large chunks; first fit trades optimality for much faster search time. |

## What's Next

This module's mechanisms (base & bounds, segmentation) share a structural weakness that free-space management (Topic 3) can only ever manage, never eliminate: they place variable-sized regions in physical memory, which is inherently prone to fragmentation. **Module 08 — Paging Fundamentals** introduces the technique that sidesteps this almost entirely: dividing both virtual and physical memory into small, fixed-size chunks (pages and page frames), so that any free frame can satisfy any page's needs — no best-fit/worst-fit decision, and dramatically less fragmentation, at the cost of a new kind of translation bookkeeping structure: the page table.
