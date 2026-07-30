# Module 09 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Translation Lookaside Buffer (TLB)** — a small, fast hardware cache of recent translations, and why locality of reference makes it effective
- [x] **TLB Misses and Context Switches** — the danger of stale cross-process entries, TLB flushing, and ASIDs as the performance-preserving alternative
- [x] **Multi-Level Page Tables** — restructuring the page table as a tree so unused regions cost nothing, at the price of more sequential lookups per TLB miss
- [x] **Inverted Page Tables** — indexing by physical frame instead of virtual page, scaling with physical memory instead of virtual address space, at the cost of needing an auxiliary hash table

## The Big Picture

This module directly answered both costs Module 08, Topic 3 quantified — one mechanism per cost, plus one refinement of the first:

```
   Module 08, Topic 3's two costs:

   TIME: every access costs 2 memory references
        │
        ▼
   Topic 1: TLB — cache recent translations, most accesses become 1 reference
        │
        ▼
   Topic 2: ASIDs — preserve that cache safely across context switches


   SPACE: flat table size ∝ entire address space, not actual usage
        │
        ▼
   Topic 3: Multi-level tables — unused regions cost one invalid directory
            entry, no second-level table allocated at all
        │
        ▼
   Topic 4: Inverted tables — a different axis entirely: size ∝ physical
            memory (system-wide), not virtual address space (per-process)
```

## Practical Connections

- **Why a program that jumps unpredictably across a huge, sparse data structure can run measurably slower than one that accesses memory sequentially, even doing the "same" amount of work** — this is TLB miss rate (Topic 1) made concrete: poor locality means fewer hits, and more full page-table walks.
- **Why virtualized environments and OSes both care about ASIDs (or equivalent tagging mechanisms) specifically as core count and process-switching frequency have grown** — Topic 2's trade-off (flush vs. tag) matters more, not less, as context switches become more frequent under modern schedulers (Modules 04–05).
- **Why 64-bit systems universally use multi-level (often 4–5 level) page tables, never a flat table** — Topic 3's core argument (a flat table for a 64-bit space would be astronomically, unusably large) is not a hypothetical; it's the direct, practical reason this design is universal today.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| TLB flush vs. ASID tagging | Flushing clears the entire TLB on every context switch (simple, always correct, discards useful cached entries). ASIDs tag entries by owning process, letting multiple processes' entries safely coexist without a full reset. |
| Multi-level page tables vs. the TLB | The TLB solves the *time* cost (extra memory reference per access) via caching. Multi-level tables solve the *space* cost (wasted entries for unused regions) via a tree structure — two different mechanisms for two different problems from Module 08, Topic 3. |
| Multi-level vs. inverted page tables | Multi-level tables are still indexed by virtual page number (just hierarchically) and sized per-process by virtual usage. Inverted tables are indexed by physical frame number, sized system-wide by physical memory, and need an auxiliary hash table for lookups. |

## What's Next

Modules 07–09 have assumed every page a process needs is always present in physical memory. **Module 10 — Beyond Physical Memory** removes that assumption: what happens when the total memory all running processes want exceeds the physical RAM actually available? It introduces swapping (moving pages temporarily out to disk) and the page-fault handling path that makes this possible — directly building on this module's page table entry fields (the present bit, previewed in Module 08, Topic 2) and setting up Module 11's page-replacement policies for deciding exactly which pages to evict when space runs short.
