# Module 07 — Address Translation

## Module Goal

By the end of this module, you will understand **the earliest, simplest mechanisms the OS and hardware use together to transparently translate a process's private virtual addresses into real physical memory locations** — directly fulfilling the transparency, efficiency, and protection goals from Module 06, Topic 1. These mechanisms (base & bounds, segmentation) are historically important and conceptually foundational, and they directly motivate paging (Module 08), the technique that superseded them.

## Topics Covered in This Module

1. **[Dynamic Relocation (Base and Bounds)](01-dynamic-relocation-base-and-bounds.md)** — The simplest possible translation scheme: one base register and one bounds register per process, and the hardware support (the Memory Management Unit) that makes it fast.
2. **[Segmentation](02-segmentation.md)** — Generalizing base & bounds to multiple independently-relocatable segments per process, fixing base & bounds' internal fragmentation problem.
3. **[Free-Space Management](03-free-space-management.md)** — How the OS (or a heap allocator, Module 06 Topic 2) tracks and satisfies variable-sized memory requests out of a pool of free space, and the classic allocation strategies (best fit, worst fit, first fit).
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 06 in full, especially Topic 1 (The Address Space Abstraction) and its three goals: transparency, efficiency, protection.

## How to Study This Module

Read in order. Topic 1 introduces the core trick every later translation mechanism in this course (including paging, Module 08) builds on: adding a small amount of hardware support so that translation happens on every memory access, fast, without OS involvement in the common case. Topic 2 shows the natural next refinement once you notice base & bounds' single biggest weakness (wasted space between the heap and stack). Topic 3 is more general-purpose — it's about managing any pool of variable-sized free space, which is directly relevant both to segmentation's per-segment placement problem and to Module 06 Topic 2's `malloc()`/`free()` heap management.
