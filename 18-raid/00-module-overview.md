# Module 18 — RAID

## Module Goal

By the end of this module, you will understand **RAID — combining multiple physical disks to act as one logical unit**, and the fundamental three-way trade-off (capacity, performance, reliability) every RAID level makes differently, covering striping (RAID 0), mirroring (RAID 1), and parity-based schemes (RAID 4/5).

## Topics Covered in This Module

1. **[Why RAID? Striping and RAID 0](01-why-raid-striping-and-raid-0.md)** — Combining disks for capacity and parallel performance, with zero reliability benefit.
2. **[Mirroring: RAID 1](02-mirroring-raid-1.md)** — Trading capacity for reliability by keeping a full duplicate copy.
3. **[Parity-Based RAID: RAID 4 and RAID 5](03-parity-based-raid-4-and-5.md)** — Getting most of mirroring's reliability at a fraction of its capacity cost, using parity.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 17 in full, especially Topic 3 (Hard Disk Drive Mechanics).

## How to Study This Module

Read in order. Topic 1 establishes the baseline: combining disks purely for capacity and speed, with no protection against a disk failing at all. Topic 2 shows the simplest possible fix (just duplicate everything), and its steep capacity cost. Topic 3 shows the clever, dominant real-world compromise (parity) that recovers most of mirroring's protection at a much smaller capacity cost. By the end, you should be able to state, for any given RAID level, exactly what it trades away and what it gains, in the same three-way currency: capacity, performance, reliability.
