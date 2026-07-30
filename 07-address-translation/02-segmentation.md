# Segmentation

## Learning Objectives

By the end of this section you should be able to:
- Explain what internal fragmentation is and how segmentation reduces it
- Explain how a segmented address is translated, including which bits select a segment
- Explain the concept of external fragmentation, and why segmentation introduces this new problem while fixing the old one

## Prerequisites

- Topic 1 (Dynamic Relocation: Base and Bounds)

## Motivation

Topic 1 ended with a clear, specific weakness: base & bounds must allocate one single contiguous physical block for a process's *entire* address space — including any unused gap between the heap and stack — wasting real memory on space nobody is using. Segmentation is the direct, natural fix: instead of one base/bounds pair per process, use several.

## Problem Statement

Recall the address space layout from Module 06, Topic 1: Code, Static Data, Heap, and Stack, with the Heap and Stack growing toward each other, leaving a variable-sized gap of currently-unused space in between. Under simple base & bounds (Topic 1), this entire range — including that unused gap — must be reserved as one contiguous physical block. If a process declares a large address space to leave room for its heap and stack to grow, but is currently only using a small fraction of it, that unused space still physically occupies real RAM, unavailable to any other process. Could the OS instead only allocate physical memory for the parts of the address space actually in use, while still supporting the illusion of one unified, contiguous virtual address space?

## Concept

### Definition

> **Segmentation** generalizes base-and-bounds translation by giving a process **multiple** independent base/bounds pairs — one for each logical **segment** of its address space (typically Code, Heap, and Stack) — rather than just one for the entire address space. Each segment can be placed independently, anywhere in physical memory, and each has its own size, so physical memory is only consumed for the parts of the address space actually in use.

### How a Segmented Address Is Translated

A virtual address under segmentation is split into two conceptual parts: a **segment number** (identifying which segment — Code, Heap, or Stack — this address refers to) and an **offset** *within* that segment. A common, simple approach uses the top few bits of the virtual address as the segment number, and the remaining bits as the offset:

```
 Virtual address (example, 2 segment-selector bits + 14 offset bits):

  ┌──┬────────────────────────────┐
  │01│      offset within segment    │
  └──┴────────────────────────────┘
   ▲
   segment number → selects which base/bounds pair to use
                     (e.g., 00 = Code, 01 = Heap, 10 = Stack)

 physical_address = segment_table[segment_number].base + offset

 (permitted only if: offset < segment_table[segment_number].bounds)
```

Each segment has its own base and bounds — held in a small hardware structure (a **segment table**) rather than a single pair of registers — and the bounds check for each segment only needs to cover that segment's actual, currently-used size, not the entire address space's maximum possible extent.

### Fixing the Fragmentation Problem

Because the Heap segment's bounds only need to cover the Heap's *actual current size* (not the maximum size it could theoretically grow to), and the Stack segment is handled entirely separately, the previously-wasted gap between Heap and Stack (Topic 1's core weakness) simply doesn't need to be allocated as physical memory at all — each segment occupies only as much physical memory as it currently, genuinely needs, placed independently, wherever convenient.

### Support for Sharing: Protection Bits

A natural, valuable side benefit of splitting the address space into distinct segments: each segment can be given its own separate protection bits (e.g., a Code segment can be marked read-and-execute-only, preventing a process from accidentally — or maliciously — overwriting its own instructions; a Heap/Stack segment can be marked read-write but not executable). This lets the OS enforce meaningfully different rules for different *kinds* of memory within the same address space — a level of nuance simple base & bounds (Topic 1), with only one undivided region, could not express at all.

## Internal Working (Preview)

```
   Segment Table (per process)                  Physical RAM

   Segment   Base     Bounds                    ┌────────────┐
   ───────  ───────  ────────                    │ (other data) │
   Code       4000      800          ────────►    ├────────────┤
   Heap      12000      300          ────┐        │ Code (4000)  │
   Stack     20000      150          ──┐  │        ├────────────┤
                                        │  └───────►│ Heap (12000) │
                                        └──────────►│ Stack (20000)│
                                                     └────────────┘

   Notice: no physical memory reserved for the "gap" between Heap
   and Stack at all — each segment is placed independently, sized
   to only what it actually currently needs.
```

## Real-World Analogy

Think of Topic 1's base & bounds as renting one enormous, single storage unit sized to hold everything you might *ever* need to store, even though right now you're only using a small fraction of it near the front and a small fraction near the back — you're still paying for (and occupying) all the unused space in between. Segmentation is like instead renting several smaller, independently-sized units at a facility — one for your "documents" (Code), one for your "furniture" (Heap), one for your "seasonal items" (Stack) — each exactly sized to what you actually currently need, placed wherever convenient in the facility, rather than one giant reserved block with a large, wasted empty middle.

## Why Segmentation Introduces External Fragmentation

Segmentation fixes internal fragmentation (wasted space *within* one reserved block, Topic 1), but introduces a related, distinct problem: because segments are variable-sized and placed independently throughout physical memory, over time, as segments are allocated and freed, physical memory can become divided into many small, scattered free chunks — none individually large enough to fit a new segment's request, even though the *total* free memory across all those scattered chunks might be more than sufficient.

> **External fragmentation** occurs when free memory becomes divided into multiple, non-contiguous chunks over time, such that no single chunk is large enough to satisfy a new request, even though the sum of all free chunks would be.

This is a direct consequence of allowing variable-sized segments to be placed and removed independently — the exact flexibility that fixed internal fragmentation is what creates the conditions for external fragmentation instead. Managing this scattered free space efficiently is the subject of the next topic, Free-Space Management.

## Advantages of Segmentation

- **Eliminates the specific internal-fragmentation waste** from Topic 1 — physical memory is only consumed for a segment's actual, current size.
- **Enables per-segment protection** — different rules (read-only, executable, etc.) can be applied to Code vs. Heap vs. Stack independently.
- **Enables sharing** — a segment containing code that many processes use identically (e.g., a common system library) can, in principle, be mapped into multiple processes' segment tables simultaneously, saving physical memory.

## Disadvantages of Segmentation

- **External fragmentation** — as segments of varying sizes are allocated and freed over time, physical memory can become scattered into chunks too small individually to satisfy new requests, even with sufficient total free space.
- **More complex hardware and OS bookkeeping** — a segment table (multiple base/bounds pairs) is more hardware and OS logic than Topic 1's single pair, and the OS must actively manage placement and fragmentation (Topic 3) rather than relying on one simple, fixed allocation.
- **Coarse granularity** — segments still correspond to entire logical regions (all of Code, all of Heap) rather than small, uniform units — a limitation that paging (Module 08) addresses by using small, fixed-size chunks instead.

## Best Practices

- When explaining segmentation, always frame it explicitly as "base & bounds, but more than one pair, one per logical region" — this framing makes clear it's a direct generalization, not an unrelated new idea.
- Keep internal fragmentation (Topic 1's problem: wasted space *inside* one reserved block) and external fragmentation (this topic's new problem: wasted space *between* scattered free chunks) as clearly distinct concepts — they're easy to conflate, but they have different causes and different mitigations (Topic 3).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Segmentation eliminates fragmentation entirely." | It eliminates the *internal* fragmentation problem from base & bounds (Topic 1), but introduces a different problem, external fragmentation — free memory becoming scattered into chunks too small individually to satisfy new requests. |
| "Every process's segments are the same fixed size." | Segments are variable-sized, reflecting each logical region's actual current needs (e.g., however large the Heap currently is) — this variability is precisely what enables eliminating internal fragmentation, and precisely what creates the conditions for external fragmentation instead. |

## Interview Questions

1. **Q: How does segmentation improve on simple base-and-bounds translation?**
   A: Instead of one base/bounds pair for a process's entire address space, segmentation uses multiple independent base/bounds pairs — one per logical segment (Code, Heap, Stack) — so physical memory is only allocated for each segment's actual current size, eliminating the internal fragmentation from unused gaps (like the space between Heap and Stack) that plain base & bounds wastes.

2. **Q: How is a segmented virtual address translated to a physical address?**
   A: A portion of the virtual address (often the top bits) identifies which segment it belongs to, selecting that segment's specific base/bounds pair from a segment table; the remaining bits are the offset within that segment, added to the selected base (after a bounds check against that segment's specific size).

3. **Q: What is external fragmentation, and why does segmentation introduce it?**
   A: External fragmentation is when free physical memory becomes divided into multiple small, non-contiguous chunks, none individually large enough for a new request, even though their total would suffice. It arises specifically because segmentation allows variable-sized segments to be placed and removed independently throughout physical memory over time, scattering free space unpredictably.

## Summary

- Segmentation generalizes base & bounds to multiple independent base/bounds pairs, one per logical segment (Code, Heap, Stack), directly eliminating the internal-fragmentation waste of Topic 1's single-region approach.
- A segmented address splits into a segment number (selecting a base/bounds pair from a segment table) and an offset within that segment.
- Segmentation also enables per-segment protection bits and sharing, but introduces external fragmentation as variable-sized segments are allocated and freed over time.
- The next topic, Free-Space Management, covers the classic strategies (best fit, worst fit, first fit) for managing exactly this kind of scattered free-space problem.
