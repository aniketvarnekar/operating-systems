# Module 10 — Beyond Physical Memory

## Module Goal

By the end of this module, you will understand **the mechanism that lets the total memory demanded by all running processes exceed the physical RAM actually installed on a machine** — swap space, the present bit, and the full page-fault handling path. This module covers the *mechanism* only (how a missing page gets brought back in); Module 11 covers the *policy* (which page to evict first when physical memory is completely full).

## Topics Covered in This Module

1. **[Swap Space and the Present Bit](01-swap-space-and-the-present-bit.md)** — Where a temporarily-evicted page actually goes, and the page table entry field that tracks whether a page is currently resident in RAM at all.
2. **[The Page Fault Handling Path](02-the-page-fault-handling-path.md)** — The complete, step-by-step sequence the OS follows when a process accesses a page that isn't currently in physical memory.
3. **[Module Summary](03-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 08, Topic 2 (Page Tables and Page Table Entries) — specifically, the present bit, previewed there
- Module 09 in full — this module assumes fluency with the TLB and page tables as the translation machinery being extended here

## How to Study This Module

Read in order. Topic 1 introduces the two pieces of infrastructure this entire module depends on: swap space (the disk-based overflow area) and the present bit (the page table entry field that distinguishes "this page is in RAM" from "this page has been moved to disk"). Topic 2 is the payoff — the complete mechanical sequence, a **page fault**, that the OS follows when a process touches a page that isn't currently resident. Pay close attention to how this deliberately reuses the trap mechanism from Module 03 — a page fault is, mechanically, just another kind of trap, handled through the exact same trap-table infrastructure you already learned.
