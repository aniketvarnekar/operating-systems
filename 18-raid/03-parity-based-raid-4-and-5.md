# Parity-Based RAID: RAID 4 and RAID 5

## Learning Objectives

By the end of this section you should be able to:
- Explain what parity is and how it can reconstruct a single missing block
- Calculate RAID 4/5's usable capacity given N disks
- Explain RAID 4's write bottleneck and how RAID 5 fixes it

## Prerequisites

- Topic 2 (Mirroring: RAID 1)

## Motivation

Topic 2 showed the simplest fix for reliability — full duplication — at a steep 50% capacity cost. This topic covers a cleverer alternative: using mathematical parity to reconstruct a single missing disk's data, at a dramatically smaller capacity cost than mirroring, and the specific write-performance problem this introduces along the way.

## Problem Statement

Mirroring (Topic 2) protects against a single disk failure by paying for a complete, redundant copy of every byte of data — doubling total storage need for the same amount of usable capacity. Is there a way to protect against a single disk failure using **less** redundant information than a full duplicate — reconstructing the missing data mathematically instead of simply reading a stored copy?

## Concept

### Parity: A Compact Redundancy Scheme

> **Parity** is a single extra value, computed from a set of data blocks (typically via the XOR — exclusive-or — operation), stored alongside them, such that if **any one** of the original data blocks (or the parity block itself) is lost, its value can be **reconstructed** from the remaining blocks plus the parity value.

The key mathematical property: XOR-based parity requires only **one extra block's worth of storage**, regardless of how many data blocks it protects — a dramatic improvement over mirroring's "one extra full copy per protected block."

```
   Data:    Block A    Block B    Block C
   Parity:  P = A XOR B XOR C

   If Block B is LOST, it can be reconstructed:
       B = A XOR C XOR P
```

### RAID 4: Dedicated Parity Disk

> **RAID 4** stripes data across N−1 disks (exactly like RAID 0), and dedicates **one additional disk entirely to parity** — storing the XOR of the corresponding blocks from every data disk.

```
   Disk 1 (data): Block 0
   Disk 2 (data): Block 1
   Disk 3 (data): Block 2
   Disk 4 (parity): Block 0 XOR Block 1 XOR Block 2
```

- **Capacity**: with N disks, N−1 disks' worth of capacity is usable — a single disk's worth of overhead, **regardless of how many total disks are in the array** — dramatically better than mirroring's 50% cost, especially as N grows larger.
- **Reliability**: any **single** disk (a data disk, or even the parity disk itself) can fail, and its contents can be fully reconstructed from the remaining disks plus the parity information.

### RAID 4's Write Bottleneck

Here's RAID 4's specific, serious weakness: **every single write to any data disk requires also updating the parity disk** (recomputing and rewriting the XOR), since the parity value must always stay correct and current. Because there is only **one** parity disk for the entire array, **every write to any data disk in the array** must also write to that same single parity disk — turning the parity disk into a serialization bottleneck: no two writes to different data disks can have their parity updates happen in true parallel, since they're both fighting over the same one physical parity disk.

### RAID 5: Rotating Parity

> **RAID 5** fixes RAID 4's bottleneck with one specific change: instead of dedicating one fixed disk entirely to parity, it **rotates** which physical disk holds the parity block for each stripe, distributing parity blocks evenly across **all** disks in the array.

```
   Stripe 1:  Disk1=A     Disk2=B     Disk3=C     Disk4=Parity(A,B,C)
   Stripe 2:  Disk1=D     Disk2=E     Disk3=Parity(D,E,F)   Disk4=F
   Stripe 3:  Disk1=Parity(G,H,I)   Disk2=G   Disk3=H    Disk4=I
```

Because parity is spread across every disk rather than concentrated on one, writes to different stripes' parity blocks can now genuinely be spread across different physical disks, relieving the single-disk bottleneck RAID 4 suffered — different concurrent writes to different stripes no longer all compete for the same one dedicated parity disk.

## Internal Working (Preview)

```
   RAID 4: ONE dedicated parity disk — BOTTLENECK

     Disk 1 (data)   Disk 2 (data)   Disk 3 (data)   Disk 4 (PARITY — every
                                                        single write updates
                                                        THIS one disk, no
                                                        matter which data
                                                        disk was written to)


   RAID 5: parity ROTATES across all disks — NO single bottleneck

     Stripe 1: D1=data  D2=data  D3=data  D4=PARITY
     Stripe 2: D1=data  D2=data  D3=PARITY  D4=data
     Stripe 3: D1=PARITY  D2=data  D3=data  D4=data

     (different stripes' parity updates land on DIFFERENT physical
      disks — writes can proceed more in parallel)


   Capacity (both RAID 4 and RAID 5), N disks: (N-1)/N usable
   — e.g., 5 disks → 4/5 usable, MUCH better than RAID 1's 1/2
```

## Real-World Analogy

Think of parity like a simple checksum digit added to a set of numbers, such that if any single one of the original numbers is lost, it can be recalculated from the others plus the checksum — you only need **one** extra digit, no matter how many original numbers you're protecting, unlike keeping a full duplicate set of every number. RAID 4 is like having one single, dedicated clerk responsible for recalculating and updating that checksum every time **any** number in the entire set changes — that one clerk becomes a bottleneck, since every update anywhere has to go through them specifically. RAID 5 is like instead rotating checksum duty among all the clerks, so that different updates to different parts of the records get their checksums recalculated by different people at the same time, rather than everyone lining up behind one single dedicated checksum clerk.

## Why This Design Is the Dominant Real-World Compromise

Parity-based RAID recovers the overwhelming majority of mirroring's reliability benefit (surviving any single disk's failure) while paying a dramatically smaller capacity cost that **improves as the array grows larger** (always exactly one disk's worth of overhead, regardless of N) — a much better trade-off than mirroring's fixed 50% cost for large arrays. RAID 5's rotation fix is a small, elegant change that removes RAID 4's single serialization bottleneck almost for free, which is precisely why RAID 5 (rather than RAID 4) became the more commonly deployed parity-based standard in real-world systems.

## Advantages of RAID 4/5

- **Much better capacity efficiency than mirroring**, especially for larger arrays — always exactly one disk's worth of overhead, regardless of total disk count.
- **Survives any single disk failure**, fully reconstructing the lost disk's data from parity and the remaining disks.
- **RAID 5 specifically avoids RAID 4's single-parity-disk write bottleneck** by rotating parity across all disks.

## Disadvantages of RAID 4/5

- **Only survives a single disk failure** — if a second disk fails before the first is repaired/reconstructed, data is lost (unlike some more advanced schemes with additional parity, not covered here, that tolerate more simultaneous failures).
- **Writes still cost more than a simple data write** — updating parity correctly requires reading the old data and old parity, computing the new parity, and writing both the new data and new parity, a genuinely more involved operation than RAID 0's direct, uncoordinated writes.
- **Reconstruction after a failure is itself a real, resource-intensive process** — rebuilding a replaced disk's contents from parity and the remaining disks takes real time and I/O bandwidth, during which the array typically has reduced (or temporarily zero) additional fault tolerance.

## Best Practices

- Prefer RAID 5 over RAID 4 in essentially all cases where a single dedicated parity disk isn't specifically required for some other reason — the rotation fix removes a real bottleneck at essentially no cost.
- When evaluating RAID levels for a specific workload, weigh capacity efficiency (Topic 1's RAID 0, and this topic's RAID 4/5) against write cost and failure-tolerance needs (Topic 2's RAID 1) explicitly, rather than defaulting to one level without considering the actual workload's read/write balance and reliability requirements.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Parity-based RAID requires as much redundant storage as mirroring." | Parity requires only one extra disk's worth of storage, regardless of how many data disks it protects, dramatically less than mirroring's fixed 50% overhead — especially for larger arrays. |
| "RAID 4 and RAID 5 have the same performance characteristics, just different names." | RAID 4 concentrates all parity on one dedicated disk, creating a write bottleneck since every write updates that same disk. RAID 5 rotates parity across all disks, relieving this bottleneck — a real, practical performance difference. |

## Interview Questions

1. **Q: What is parity, and how does it protect against a single disk's failure using less storage than mirroring?**
   A: Parity is a single value computed (typically via XOR) from a set of data blocks, such that any one lost block (or the parity block itself) can be reconstructed from the rest. It requires only one extra block's worth of storage regardless of how many data blocks it protects, unlike mirroring's full duplicate copy.

2. **Q: What is RAID 4's write bottleneck, and how does RAID 5 fix it?**
   A: RAID 4 dedicates one fixed disk entirely to parity, so every write to any data disk also requires updating that same single parity disk, serializing all writes through it. RAID 5 rotates which physical disk holds parity for each stripe, distributing parity updates across all disks so different writes don't all compete for the same one disk.

3. **Q: What is the usable capacity of a RAID 4 or RAID 5 array with N disks?**
   A: (N−1)/N of the total raw capacity — exactly one disk's worth of overhead is spent on parity, regardless of how large N is, making the capacity efficiency improve as more disks are added, unlike mirroring's fixed 50% cost.

## Summary

- Parity, computed via XOR across a set of data blocks, allows any single lost block to be reconstructed using only one extra block's worth of storage — far more capacity-efficient than mirroring.
- RAID 4 dedicates one disk entirely to parity, creating a write bottleneck since every write updates that same disk.
- RAID 5 rotates parity across all disks, relieving this bottleneck, and is the more commonly deployed real-world standard as a result.
- This closes out the module's coverage of RAID — the module summary ties striping, mirroring, and parity together as three distinct trade-off points before Module 19 introduces the file and directory abstractions that sit on top of whatever physical storage (single disk or RAID array) actually underlies them.
