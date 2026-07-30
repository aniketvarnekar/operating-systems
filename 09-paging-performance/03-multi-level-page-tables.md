# Multi-Level Page Tables

## Learning Objectives

By the end of this section you should be able to:
- Explain why a flat page table wastes space on unused regions of an address space
- Explain how a multi-level page table avoids this waste, using a concrete two-level example
- Explain the trade-off multi-level page tables make: less space, at the cost of more translation steps

## Prerequisites

- Module 08, Topic 3 (The Cost of Paging) — specifically, the space-cost problem
- Topic 1 (The Translation Lookaside Buffer) — helpful context, not required

## Motivation

Topics 1–2 solved the *time* half of Module 08, Topic 3's cliffhanger (the TLB). This topic solves the *space* half: a flat, single-level page table's size depends on the entire virtual address space's size, not on how much of it a process actually uses — wasteful, since most real processes use only a small fraction of their theoretical address space (Module 06, Topic 1's heap/stack gap, yet again).

## Problem Statement

Recall Module 08, Topic 3's calculation: a flat page table for a 4 GB address space with 4 KB pages needs roughly 1 million entries — 4 MB of table, per process — regardless of whether that process actually uses 4 GB or just a few hundred kilobytes of it. Most of those million entries are simply marked invalid (Module 08, Topic 2), representing regions of the address space the process never touches at all (like the large gap between the heap and stack). Is there a way to avoid paying for page table entries covering address-space regions that are never actually used?

## Concept

### The Core Idea: A Tree of Page Tables

> A **multi-level page table** restructures the flat page table into a tree: a single top-level structure (the **page directory**) holds pointers to smaller, second-level page tables, and each second-level table holds the actual page table entries (Module 08, Topic 2) for one specific chunk of the address space. Critically, **if an entire chunk of the address space is completely unused, the page directory entry pointing to its corresponding second-level table can simply be marked invalid — and that entire second-level table never needs to be allocated in memory at all.**

This is the key insight: instead of reserving space for every single possible page's entry up front (as a flat table does), a multi-level table only allocates the smaller, second-level tables that actually correspond to regions the process is using — entire chunks of unused address space (like the heap/stack gap) cost essentially nothing, since their entire second-level table is simply never created.

### Splitting the Virtual Address Into Three Parts

Under a two-level page table, the virtual address is split into three parts instead of the flat table's two (Module 08, Topic 2):

```
 Flat (single-level) page table address split:

  ┌─────────────────────┬──────────────┐
  │         VPN             │    Offset      │
  └─────────────────────┴──────────────┘

 Two-level page table address split:

  ┌───────────┬───────────┬──────────────┐
  │  Page Dir.  │  Page Table  │    Offset      │
  │   Index      │   Index       │                │
  └───────────┴───────────┴──────────────┘
```

- The **Page Directory Index** selects an entry in the top-level page directory, which either points to a second-level page table (if that region is in use) or is marked invalid (if the entire region is unused — no second-level table exists at all for it).
- The **Page Table Index** then selects the actual page table entry within that specific second-level table.
- The **Offset** works exactly as it did under flat paging (Module 08, Topic 2) — the byte position within the page, passed through unchanged.

### Worked Example: Why This Saves Space

Suppose a process uses only its Code, a small Heap, and a small Stack, leaving the vast middle of its address space completely untouched. Under a flat table, every single page in that untouched middle region still needs an entry (marked invalid) in the one giant table. Under a two-level table, the **entire untouched middle region can correspond to just a handful of page-directory entries, each marked invalid — with no second-level tables ever allocated for them at all.** The memory saved is exactly the size of all those never-created second-level tables — potentially the overwhelming majority of a flat table's total size, for a typical, sparsely-used address space.

## Internal Working (Preview)

```
   Page Directory (top level)              Second-Level Page Tables
   ┌─────┬────────────────┐               (only allocated for IN-USE regions)
   │ Idx  │ Valid │ Points to │
   ├─────┼──────┼──────────┤
   │  0    │  1     │  ──────────┼──────►  ┌───────────────┐
   │  1    │  1     │  ──────────┼──────►  │ PTEs for Code    │
   │  2    │  0     │   (none)     │        └───────────────┘
   │  3    │  0     │   (none)     │        ┌───────────────┐
   │  ...  │  0     │   (none)     │        │ PTEs for Heap    │
   │  N-1  │  1     │  ──────────┼──────►  └───────────────┘
   └─────┴──────┴──────────┘                ┌───────────────┐
                                              │ PTEs for Stack   │
     Entries marked 0 (invalid) have NO       └───────────────┘
     corresponding second-level table
     allocated in memory AT ALL — this
     is exactly where the space savings
     come from, compared to a flat table
     that would need real entries (even
     invalid ones) for every one of
     these unused regions.
```

## Real-World Analogy

Think of a flat page table like a massive city phone directory that prints one line for *every possible* phone number in the entire numbering system — including millions of numbers that were never actually assigned to anyone — just so the format stays uniform and simple to look up. A multi-level page table is like organizing that same directory hierarchically instead: a slim master index lists only the specific area codes that actually have any assigned numbers at all, and only for *those* area codes does a full, detailed sub-directory get printed and bound at all — area codes with zero assigned numbers simply don't get a sub-directory printed in the first place, saving enormous amounts of paper (memory) compared to printing every theoretically possible number.

## Why This Design Is Necessary

A flat page table's size is fundamentally tied to the *size of the address space*, regardless of actual usage — exactly the problem Module 08, Topic 3 quantified. Multi-level page tables tie the *actual allocated* table size to how much of the address space is genuinely in use instead, by allowing entire unused regions to skip having any second-level table at all. This is a direct, structural fix, not a workaround — it changes what determines page-table memory cost from "how big could this address space theoretically be" to "how much of it does this process actually use."

## Advantages of Multi-Level Page Tables

- **Dramatically reduced memory overhead** for typical, sparsely-used address spaces — unused regions cost only a single invalid page-directory entry each, rather than an entire range of individually-invalid flat-table entries.
- **Scales naturally with actual usage**, rather than with the theoretical maximum size of the address space.

## Disadvantages / Costs

- **More translation steps per TLB miss** — a two-level table requires two sequential memory reads (first the page directory, then the appropriate second-level table) to complete a translation, compared to a flat table's single read; deeper hierarchies (three or more levels, used for very large modern address spaces) require correspondingly more sequential reads. This cost is specifically paid only on a TLB miss (Topic 1) — a TLB hit still resolves in one step regardless of how many page-table levels exist underneath.
- **More complex OS bookkeeping** — the OS must manage allocating and freeing second-level tables dynamically as regions of an address space come into and go out of use, rather than working with one simple, fixed-size flat table.

## Best Practices

- When explaining why modern 64-bit systems use multi-level (often 4- or 5-level) page tables rather than a flat table, lead directly with this topic's core trade-off: a flat table for a 64-bit address space would be astronomically large, while a multi-level table's actual memory cost scales with real usage instead.
- Keep this topic's cost (extra sequential memory reads per TLB miss) distinct from Topic 1's TLB hit/miss cost — they compound (a multi-level TLB miss costs even more than a flat-table TLB miss would), but they're solving two different problems (space vs. time) with two different mechanisms.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Multi-level page tables make translation faster." | They make translation require *more* sequential steps on a TLB miss (one read per level), not fewer — their benefit is purely in *space* savings for sparsely-used address spaces, not translation speed. The TLB (Topics 1–2) is what addresses speed. |
| "A two-level page table still needs an entry for every possible page, just organized differently." | The entire point is that entirely unused regions of the address space correspond to invalid page-directory entries with NO second-level table allocated at all — those pages' individual entries genuinely never exist in memory, unlike a flat table where every page gets an entry regardless of use. |

## Interview Questions

1. **Q: What specific problem do multi-level page tables solve?**
   A: A flat, single-level page table's size scales with the entire virtual address space's size, regardless of actual usage, wasting memory on entries for entirely unused regions. Multi-level page tables let unused regions correspond to a single invalid page-directory entry, with no second-level table ever allocated for them, so table size scales with actual usage instead.

2. **Q: How does a two-level page table translate a virtual address?**
   A: The address splits into a page directory index, a page table index, and an offset. The page directory index selects an entry that either points to a second-level table (if that region is in use) or is invalid (if unused); if valid, the page table index then selects the actual entry within that second-level table, and the offset passes through unchanged.

3. **Q: What's the cost of using a multi-level page table instead of a flat one?**
   A: More sequential memory reads are needed to complete a translation on a TLB miss — one per level of the table hierarchy — compared to a flat table's single read. This cost only applies on a TLB miss; a TLB hit (Topic 1) still resolves in one step regardless of the underlying table structure.

## Summary

- A flat page table's size scales with the full address space size, wasting memory on entries for entirely unused regions — the space-cost problem from Module 08, Topic 3.
- A multi-level page table restructures this as a tree: unused regions correspond to invalid page-directory entries with no second-level table ever allocated, saving substantial memory for typical, sparsely-used address spaces.
- The trade-off is more sequential memory reads per TLB miss (one per table level), though this doesn't affect TLB hits at all.
- The next topic covers inverted page tables, a fundamentally different structure that solves the same space problem by scaling with physical memory size instead of virtual address space size.
