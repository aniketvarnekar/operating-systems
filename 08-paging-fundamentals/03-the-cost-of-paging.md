# The Cost of Paging

## Learning Objectives

By the end of this section you should be able to:
- Calculate the size of a naive, single-level page table given address space size, page size, and PTE size
- Explain why this memory cost is paid per process, not once for the whole system
- Explain why every memory access under paging requires an extra memory reference, and what that costs

## Prerequisites

- Topic 2 (Page Tables and Page Table Entries)

## Motivation

Topic 1 sold paging as an elegant fix for fragmentation; Topic 2 showed the data structure that makes it work. This topic is the deliberate reality check: quantifying, with real numbers, exactly how expensive a naive implementation of that data structure actually is — expensive enough that no real system uses the naive version unmodified, directly motivating Module 09.

## Problem Statement

A page table (Topic 2) needs one entry for *every possible* virtual page in a process's address space — not just the pages actually in use. If a process's address space is large, how large does its page table itself need to be, just to hold all those entries (even the ones marked invalid)? And separately: if every single memory access requires first consulting the page table to translate the address, doesn't that mean every memory access now costs *at least two* real memory references — one to read the page table entry, and one for the actual access itself?

## Concept

### Quantifying Page Table Size

Suppose a system uses 32-bit virtual addresses (a 4 GB address space) with 4 KB pages, and each page table entry (Topic 2) is 4 bytes:

- Page size = 4 KB = 2^12 bytes → the offset portion of the virtual address is 12 bits.
- Total virtual address space = 2^32 bytes → the remaining 32 − 12 = 20 bits form the Virtual Page Number.
- Number of possible pages = 2^20 ≈ 1 million pages.
- Page table size = (number of pages) × (PTE size) = 2^20 × 4 bytes = 4 MB.

**A single process's page table alone would need 4 MB** — just for the translation bookkeeping, before counting a single byte of the process's actual code, data, or stack. This is per **process**: a system running 100 such processes would need roughly 400 MB just for page tables, even if most processes are using only a tiny fraction of their full 4 GB address space (recall from Topic 2 that most of those entries would be marked invalid, representing address-space regions never actually used — but the flat table still needs to reserve space for the possibility, in this naive design).

### The Extra-Memory-Reference Problem

Separately from the space cost, there's a real time cost: translating **any** virtual address first requires reading the corresponding page table entry from memory (to find the physical frame number), and only *then* can the actual, intended memory access proceed. Naively, this means **every single memory reference a process makes now costs two real memory accesses instead of one** — a potential doubling of memory-access time overall, which would be a severe, unacceptable performance regression compared to base & bounds' single-addition translation (Module 07, Topic 1).

## Internal Working (Preview)

```
 Naive single-level page table size, per process:

   virtual address space size ÷ page size = number of virtual pages
   number of virtual pages × PTE size     = page table size

   Example: 4 GB address space, 4 KB pages, 4-byte PTEs
        4 GB / 4 KB = 2^20 pages
        2^20 pages × 4 bytes/PTE = 4 MB  ← per process, regardless of
                                            how much of that 4 GB the
                                            process actually uses


 Naive translation cost, per memory access:

   Step 1: read page table entry from memory  (extra access)
   Step 2: read/write the actual target data  (the access that was wanted)

   → 2 memory accesses instead of 1, for EVERY reference
```

## Real-World Analogy

Imagine a massive company directory that must reserve one entry for every possible employee ID number the company's numbering scheme could ever assign — say, IDs 1 through 1,000,000 — even if the company currently only employs 200 people. The directory (the page table) still physically occupies space proportional to the full range of *possible* IDs, not just the *actual* headcount, because there was no way to know in advance which specific IDs would ever be used. And separately: every time anyone wants to actually reach a specific employee, they must first look up that employee's current office location in this directory (one lookup), and *then* actually walk to that office (the real, intended action) — two separate steps for something that, without the directory layer, would have been one direct action.

## Why This Cost Is a Serious, Real Problem

Both costs identified here — a multi-megabyte-per-process space overhead, and a doubling of per-access memory-reference time — are severe enough that no practical, real-world OS uses a naive, flat, single-level page table exactly as described in Topic 2 without further refinement. The space cost scales with the *size of the address space*, not with how much of it a process actually uses, which is wasteful precisely because most real processes use only a small fraction of their full theoretical address space (Module 06, Topic 1's heap/stack gap, again). The time cost, if left unaddressed, would erase much of paging's appeal by making ordinary memory access dramatically slower than it was under simpler schemes.

## Advantages of Confronting This Cost Directly

- **Motivates real, load-bearing engineering** — understanding precisely *why* naive paging is too expensive is what makes Module 09's multi-level page tables (attacking the space problem) and TLBs (attacking the time problem) make sense as necessary, not optional, refinements.
- **Reusable quantitative reasoning** — the "address space size ÷ page size × PTE size" calculation used here is a template you can reapply to evaluate any specific page-table design's actual memory footprint.

## Disadvantages Being Set Up for Module 09

- **Space**: a flat, single-level page table's size depends only on the *total* virtual address space size, not on how much of it is actually used — directly motivating multi-level page tables (Module 09), which avoid allocating table space for entirely unused regions.
- **Time**: every memory access paying for an extra page-table memory reference is a severe, unacceptable slowdown if left unaddressed — directly motivating the TLB (Module 09), a small, fast hardware cache specifically for recently-used translations.

## Best Practices

- When evaluating any address-translation scheme, always separately account for both its **space cost** (how much memory the translation bookkeeping itself consumes) and its **time cost** (how many actual memory references a single logical access requires) — as this topic shows, a design can look elegant on one axis while being seriously impractical on the other.
- Practice the page-table-size calculation (address space size ÷ page size × PTE size) directly — it's a common, concrete way this material is tested, and it builds real intuition for why page size and address space size matter so much to overall system design.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A page table's size depends on how much memory the process is actually using." | Under a naive, flat, single-level page table, its size depends on the size of the *entire possible* virtual address space, regardless of how much of it any given process actually uses — even mostly-unused address spaces still need a full-sized table under this design. |
| "Paging has no extra runtime cost compared to base & bounds, since it's just an array lookup." | Every single memory access under naive paging requires an additional memory reference (to read the page table entry) before the actual intended access — a real, serious performance cost if not specifically addressed (as Module 09 does). |

## Interview Questions

1. **Q: For a 32-bit address space with 4 KB pages and 4-byte page table entries, roughly how large is a single process's flat page table?**
   A: About 4 MB — the address space has 2^20 possible pages (2^32 ÷ 2^12), and each needs a 4-byte entry, giving 2^20 × 4 bytes = 4 MB, regardless of how much of that address space the process actually uses.

2. **Q: Why is a naive flat page table's size described as wasteful for most real processes?**
   A: Because its size depends on the size of the entire possible address space, not on how much of it is actually used — most processes use only a small fraction of their theoretical address space, so most page table entries end up marked invalid, yet still consume space in a flat table.

3. **Q: Why does naive paging risk doubling the cost of every memory access?**
   A: Because translating a virtual address first requires reading its page table entry from memory (one memory access), before the actual intended memory access can proceed (a second memory access) — turning every single logical reference into two physical ones if left unaddressed.

## Summary

- A naive, flat, single-level page table's size scales with the full virtual address space size, not with actual usage, and can easily reach several megabytes per process.
- Every memory access under naive paging requires an extra memory reference just to read the page table entry, effectively doubling memory-access cost if unaddressed.
- Both costs are severe enough that no practical OS uses this naive design unmodified — they directly motivate Module 09's two refinements: multi-level page tables (attacking the space problem) and the TLB (attacking the time problem).
- This closes out the module's core paging concepts — the module summary ties fixed-size chunking (Topic 1), the page table structure (Topic 2), and this cost analysis (Topic 3) together before Module 09 addresses both weaknesses directly.
