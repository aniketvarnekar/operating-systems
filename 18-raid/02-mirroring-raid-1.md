# Mirroring: RAID 1

## Learning Objectives

By the end of this section you should be able to:
- Explain how RAID 1 (mirroring) achieves fault tolerance
- Calculate RAID 1's usable capacity given N disks
- Explain RAID 1's read and write performance characteristics

## Prerequisites

- Topic 1 (Why RAID? Striping and RAID 0)

## Motivation

Topic 1 ended with RAID 0's complete lack of reliability — any single disk failing destroys everything. This topic covers the most direct, simplest possible fix: instead of spreading data thinly across disks with no backup, keep a full, complete duplicate of everything.

## Problem Statement

If losing a single disk under RAID 0 destroys the entire array, what is the most straightforward way to ensure that losing one disk **doesn't** lose any data at all? What would it cost in exchange?

## Concept

### Definition

> **RAID 1 (mirroring)** stores an identical, complete copy of every piece of data on **two (or more) separate disks** — every write goes to both (or all) mirrored disks simultaneously, so that if any one disk in a mirrored pair fails, the exact same data is still fully intact on the other.

```
   Disk 1 (primary):  [Block 0][Block 1][Block 2][Block 3]
   Disk 2 (mirror):    [Block 0][Block 1][Block 2][Block 3]
                        (an EXACT duplicate, kept in sync on every write)
```

### Capacity Cost

With N disks organized as N/2 mirrored pairs (the simplest configuration, two disks per pair), only **half** of the total raw capacity is usable — the other half is entirely consumed by the duplicate copies. This is a steep, direct capacity cost in exchange for surviving a disk failure: RAID 1 with 2 disks of capacity C each yields only C usable capacity, not 2C.

### Performance Characteristics

- **Reads** can actually be **faster** than a single disk in some implementations: since the same data exists on multiple disks, read requests can be split across them (e.g., one mirror handles the first half of a batch of reads, the other handles the second half), similar in spirit to RAID 0's parallel read benefit.
- **Writes** are **not** faster, and can be marginally slower: every single write must be applied to **all** mirrored copies before it's considered complete, so write throughput is limited to roughly what a single disk can do (all mirrors must complete, not just one), plus whatever small coordination overhead is needed to keep them synchronized.

### Reliability

RAID 1 directly, straightforwardly solves Topic 1's core problem: if one disk in a mirrored pair fails, the surviving disk still holds a complete, correct copy of all the data — the array continues operating correctly (typically with reduced redundancy, since the failed disk's mirror is temporarily gone, until it's replaced and re-synchronized), with **no data loss** from that single failure.

## Internal Working (Preview)

```
   RAID 1, 2 disks, each capacity C:

   Disk 1: [Block 0][Block 1][Block 2][Block 3]  ── FULL copy
   Disk 2: [Block 0][Block 1][Block 2][Block 3]  ── FULL copy (mirror)

   Capacity:    C usable (HALF of the raw 2C total — the other C
                is "spent" entirely on the duplicate)
   Performance: reads can be split across both disks (faster);
                writes must complete on BOTH disks (no faster
                than a single disk, for writes)
   Reliability: EITHER disk can fail entirely — the other still
                holds a complete, correct copy. NO data lost.
```

## Real-World Analogy

Think of mirroring like keeping two complete, identical physical copies of an important document in two separate filing cabinets, updating both simultaneously every single time a change is made. If a fire destroys one cabinet entirely, the other cabinet still has everything, completely intact — a straightforward, easy-to-reason-about form of protection. The cost is equally straightforward: you're using twice the physical filing-cabinet space to store the same actual amount of unique information, since half of your total storage is dedicated purely to the duplicate.

## Why This Design Is Valued Despite Its Capacity Cost

RAID 1's core appeal is its sheer **simplicity** — the recovery logic is as straightforward as "just read from the surviving copy," with no complex reconstruction math needed at all, unlike the parity-based schemes in the next topic. For workloads where reliability is critical and the 50% capacity cost is an acceptable trade (e.g., a database's most critical data), RAID 1 remains a common, straightforward, well-understood choice specifically because of this simplicity.

## Advantages of RAID 1

- **Straightforward, simple fault tolerance** — survives any single disk's failure within a mirrored pair, with trivially simple recovery (just use the surviving copy).
- **Potentially faster reads** — read requests can be distributed across the mirrored copies.

## Disadvantages of RAID 1

- **Steep capacity cost** — only half of total raw capacity is usable, regardless of how many disks are mirrored together.
- **No write performance benefit** — every write must complete on all mirrored copies, capping write throughput at roughly a single disk's speed for that operation, plus synchronization overhead.

## Best Practices

- Choose RAID 1 specifically when reliability matters more than maximizing usable capacity, and when the workload's write patterns can tolerate no meaningful write-throughput improvement from the array.
- Always factor the 50% capacity cost explicitly into storage planning — it's easy to underestimate how much raw disk capacity is actually needed to achieve a given amount of *usable*, mirrored storage.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "RAID 1 with N disks provides N times the capacity, like RAID 0." | RAID 1 organizes disks into mirrored pairs (or larger mirrored groups), with only half (or 1/group-size) of total raw capacity usable — the rest is consumed entirely by duplicate copies, not contributing any additional unique storage. |
| "RAID 1 improves both read and write performance equally." | Reads can benefit from being split across mirrored copies, but writes must complete on every mirrored disk before being considered done, capping write throughput at roughly a single disk's speed, not improving it. |

## Interview Questions

1. **Q: How does RAID 1 achieve fault tolerance?**
   A: It stores an identical, complete copy of every piece of data on two or more separate disks, keeping them synchronized on every write — if any one disk fails, the surviving mirror still holds a complete, correct copy of all the data.

2. **Q: What is RAID 1's usable capacity, given N disks in mirrored pairs?**
   A: Only half of the total raw capacity — N disks yield N/2 disks' worth of usable storage, since the other half is entirely consumed by duplicate copies.

3. **Q: Why doesn't RAID 1 improve write performance the way it can improve read performance?**
   A: Every write must be applied to and completed on all mirrored copies before it's considered done, so write throughput is capped at roughly a single disk's speed (plus synchronization overhead) rather than benefiting from parallelism across disks.

## Summary

- RAID 1 (mirroring) keeps a complete, synchronized duplicate of all data on separate disks, directly solving RAID 0's complete lack of fault tolerance.
- Its capacity cost is steep: only half of total raw capacity is usable.
- Reads can benefit from parallelism across mirrors; writes cannot, since every mirror must complete each write.
- The next topic covers parity-based RAID (levels 4 and 5), which recovers most of mirroring's reliability benefit at a much smaller capacity cost, using a cleverer redundancy scheme than simple duplication.
