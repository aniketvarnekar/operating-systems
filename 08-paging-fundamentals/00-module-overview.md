# Module 08 — Paging Fundamentals

## Module Goal

By the end of this module, you will understand **paging — the dominant, modern technique for virtual-to-physical address translation** — including exactly why it sidesteps the fragmentation problems Module 07 ended on, how the page table data structure works, and what a page table entry actually contains. This module deliberately sets up a new cost (page tables can themselves be large), which Module 09 addresses directly.

## Topics Covered in This Module

1. **[The Core Idea of Paging](01-the-core-idea-of-paging.md)** — Dividing memory into small, fixed-size chunks, and why this sidesteps external fragmentation almost entirely.
2. **[Page Tables and Page Table Entries](02-page-tables-and-page-table-entries.md)** — The per-process data structure that maps virtual pages to physical frames, and exactly what a page table entry stores.
3. **[The Cost of Paging](03-the-cost-of-paging.md)** — Why naive, single-level page tables consume a surprising amount of memory, setting up Module 09's performance and space refinements.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 07 in full, especially Topic 2 (Segmentation) and its external-fragmentation weakness.

## How to Study This Module

Read in order. Topic 1 is the single most important idea in this module: fixed-size chunks instead of variable-sized ones. Once that clicks, Topic 2's page table is just "the bookkeeping structure needed to remember which chunk maps to which" — a natural, almost inevitable consequence of Topic 1's core idea. Topic 3 is the deliberate cliffhanger of the module: it shows that this elegant fix isn't free, and quantifies exactly how expensive a naive implementation would be — setting up Module 09's TLB and multi-level page table refinements as the direct answer.
