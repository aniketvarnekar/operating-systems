# Module 09 — Paging Performance

## Module Goal

By the end of this module, you will understand **the two direct engineering answers to Module 08's cliffhanger**: the TLB, a small, fast hardware cache that eliminates the extra-memory-reference cost for the vast majority of accesses, and multi-level (plus inverted) page tables, which eliminate the need to reserve table space for entirely unused regions of an address space.

## Topics Covered in This Module

1. **[The Translation Lookaside Buffer (TLB)](01-the-translation-lookaside-buffer.md)** — A small, fast hardware cache of recent virtual-to-physical translations, and why it works so well in practice.
2. **[TLB Misses and Context Switches](02-tlb-misses-and-context-switches.md)** — What happens when a translation isn't cached, and the specific problem context switches create for a shared TLB.
3. **[Multi-Level Page Tables](03-multi-level-page-tables.md)** — Restructuring the page table itself as a tree, so entirely unused regions of an address space cost nothing to represent.
4. **[Inverted Page Tables](04-inverted-page-tables.md)** — A fundamentally different structure that scales with physical memory size instead of virtual address space size.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 08 in full, especially Topic 3 (The Cost of Paging) — this module directly answers the two costs quantified there.

## How to Study This Module

Read in order. Topics 1–2 form one continuous idea about **time**: the TLB is the direct fix for "every access costs two memory references," and Topic 2 covers the sharp edge case (context switches) that a naive TLB design would get wrong. Topics 3–4 pivot to **space**: two different structural fixes for "a flat page table wastes space on entirely unused regions," each making a different trade-off. Keep Module 08, Topic 3's two numbers (page table size, and the doubled access cost) in mind throughout — every mechanism in this module exists specifically to make one of those two numbers acceptable in practice.
