# Why RAID? Striping and RAID 0

## Learning Objectives

By the end of this section you should be able to:
- Explain the three-way trade-off (capacity, performance, reliability) every RAID level makes
- Explain how striping works and why it improves performance
- Explain precisely why RAID 0 offers zero reliability benefit — and is in fact worse than a single disk

## Prerequisites

- Module 17, Topic 3 (Hard Disk Drive Mechanics)

## Motivation

A single disk has a fixed capacity, a fixed performance ceiling, and — being one physical mechanical device — a real chance of eventually failing entirely, losing everything stored on it. This topic introduces the general idea of combining multiple disks to address some of these limits, and the simplest possible combination scheme, which addresses capacity and performance but deliberately ignores reliability.

## Problem Statement

Suppose you need more storage capacity than one disk provides, or faster throughput than one disk's mechanical limits (Module 17, Topic 3) allow. Could multiple physical disks be combined and presented to the rest of the system as if they were one single, larger, faster logical disk?

## Concept

### RAID: Redundant Array of Independent Disks

> **RAID** is a general technique for combining multiple physical disks into one logical unit, managed either by dedicated hardware or by software, presenting a single, larger logical disk to the rest of the system — while internally distributing (and sometimes duplicating) data across the underlying physical disks according to a specific scheme, called a **RAID level**.

Every RAID level makes a different trade-off among three, often competing, goals:

- **Capacity**: how much of the underlying disks' total raw storage is actually usable for data, versus consumed by redundancy overhead.
- **Performance**: how much faster (or slower) the array is compared to a single disk, for reads and/or writes.
- **Reliability**: whether the array can survive one (or more) individual disk failures without losing data.

### Striping

> **Striping** spreads sequential data across multiple disks in small, fixed-size chunks, round-robin style, rather than storing it all on one disk.

```
   Data:  [ Block 0 ][ Block 1 ][ Block 2 ][ Block 3 ][ Block 4 ][ Block 5 ]

   Striped across 3 disks:

   Disk 1: Block 0, Block 3
   Disk 2: Block 1, Block 4
   Disk 3: Block 2, Block 5
```

Because consecutive blocks live on **different physical disks**, requests for those blocks can potentially be serviced **in parallel**, by different disks' independent arms and heads (Module 17, Topic 3) simultaneously — directly increasing throughput for workloads that access data across multiple stripes, compared to a single disk that could only service one request (or one sequential stream) at a time.

### RAID 0: Pure Striping, No Redundancy

> **RAID 0** applies striping across all disks in the array with **no redundant data at all** — every byte of usable capacity across all N disks contributes to the array's total storage, and none is spent on protection.

- **Capacity**: full — N disks of capacity C each yield N × C total usable capacity, with zero overhead.
- **Performance**: excellent — read/write requests can be spread across all N disks in parallel.
- **Reliability**: **worse than a single disk**, not merely "no better."

### Why RAID 0 Is Worse Than a Single Disk for Reliability

This is the specific, non-obvious, important result: with data striped across N disks, **losing any single one of the N disks corrupts the entire logical array**, since every file's data is very likely spread across multiple physical disks — a single disk's worth of every stripe is now missing. The probability that *at least one* of N independent disks fails within a given period is **higher** than the probability that one specific single disk fails in that same period (more disks, more independent chances for any one of them to fail) — so RAID 0 is not merely "no more reliable than one disk," it is measurably **more likely** to experience a data-loss event than a single disk would be, precisely because it has more physical components that could each independently fail.

## Internal Working (Preview)

```
   RAID 0 across 3 disks — striped, NO redundancy:

   Disk 1: [Block 0][Block 3]     ← if THIS disk fails,
   Disk 2: [Block 1][Block 4]        Blocks 0, 3 are GONE —
   Disk 3: [Block 2][Block 5]        and since most files span
                                       multiple stripes, essentially
                                       EVERY file is now corrupted

   Capacity:    3 disks' worth, FULL (no overhead)
   Performance: reads/writes spread across 3 disks — FAST
   Reliability: WORSE than 1 disk (3x the chances of a failure
                that destroys the WHOLE array, not just 1/3 of it)
```

## Real-World Analogy

Think of striping like splitting a single long book into three separate physical volumes, distributed page-by-page-in-rotation rather than kept as one whole chapter per volume — three readers (disks) can each work through their own volume simultaneously, reading the whole book faster together than one reader working through it alone (RAID 0's performance benefit). But now, losing just **one** of those three volumes doesn't just cost you a third of the book cleanly — since pages were distributed round-robin, that missing volume contains scattered pages from throughout the *entire* book, effectively rendering the whole book unreadable, not just one clean third of it. And with three physical volumes instead of one, there are simply three separate chances for any one of them to be lost, rather than one.

## Why This Design Is Still Sometimes Chosen Despite the Reliability Cost

RAID 0 is a legitimate, deliberate choice specifically for workloads where **capacity and raw performance genuinely matter more than reliability** — for example, temporary scratch space for a large computation, where the data is disposable and can be regenerated or is backed up elsewhere entirely, and losing it mid-computation is merely inconvenient, not catastrophic. It is a poor choice for any data whose loss would be genuinely costly, which is exactly why the next topics introduce schemes that trade away some of RAID 0's capacity or performance specifically to buy back reliability.

## Advantages of RAID 0

- **Full usable capacity** — no storage is spent on redundancy at all.
- **Excellent parallel performance** — striping lets multiple disks service requests simultaneously.

## Disadvantages of RAID 0

- **Zero fault tolerance, and measurably worse reliability than a single disk** — any one disk's failure destroys the entire logical array's data, and having more disks means more independent chances for such a failure to occur.

## Best Practices

- Reserve RAID 0 specifically for scenarios where the data is disposable, easily regenerated, or already durably backed up elsewhere — never for data whose loss would be genuinely costly.
- When asked "what does RAID 0 protect against," the correct answer is: nothing — it exists purely for capacity and performance, and actively worsens the reliability picture compared to a single disk.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "RAID 0 provides at least some redundancy, since it's part of the RAID family." | RAID 0 stores zero redundant data — the "R" in RAID (redundant) does not apply to this specific level at all; it exists purely to combine disks for capacity and performance. |
| "RAID 0's reliability is the same as a single disk's, since you're just splitting data across more disks." | It's measurably worse: any one of the N disks failing destroys the entire array, and more disks means more independent opportunities for at least one failure to occur within a given period — a higher, not equal, risk compared to one single disk. |

## Interview Questions

1. **Q: What is striping, and how does it improve performance?**
   A: Striping spreads sequential data across multiple disks in small, fixed-size chunks, round-robin style. Because consecutive blocks live on different physical disks, requests spanning multiple stripes can be serviced in parallel by different disks simultaneously, increasing throughput compared to a single disk.

2. **Q: Why is RAID 0 considered worse for reliability than a single disk, not merely "no better"?**
   A: Because data is striped across all disks with no redundancy, any single disk's failure corrupts the entire array (since most files span multiple disks). With N disks, there are N independent chances for a failure, making a data-loss event more likely overall than with just one disk.

3. **Q: What are the three trade-off dimensions every RAID level balances differently?**
   A: Capacity (usable storage versus redundancy overhead), performance (throughput relative to a single disk), and reliability (whether the array survives an individual disk failure).

## Summary

- RAID combines multiple physical disks into one logical unit, with every RAID level trading off capacity, performance, and reliability differently.
- Striping spreads data across disks in small chunks, enabling parallel access and improved performance.
- RAID 0 applies pure striping with no redundancy — full capacity, excellent performance, but reliability measurably worse than a single disk, since any one disk's failure destroys the entire array.
- The next topic covers RAID 1 (mirroring), the simplest possible fix for RAID 0's complete lack of reliability.
