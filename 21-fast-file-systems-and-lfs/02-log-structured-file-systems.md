# Log-Structured File Systems

## Learning Objectives

By the end of this section you should be able to:
- Explain the core idea of a log-structured file system: buffering writes and flushing them sequentially
- Explain why LFS needs a way to locate the most recent inode for a given file, and how the imap addresses this
- Explain the cleaning problem LFS introduces, and why it's a necessary consequence of the design

## Prerequisites

- Topic 1 (FFS and the Locality Principle)
- Module 20, Topic 1 (On-Disk Structures: Inodes and Bitmaps)

## Motivation

Topic 1's FFS clusters related data to *reduce* seek costs, but still performs many separate, small, scattered writes for ordinary file operations (Module 20, Topic 3's trace shows updating an inode here, a directory entry there, a data block somewhere else). This topic covers a more radical idea: what if, instead of merely clustering writes, a file system converted essentially *all* writes into one single, purely sequential stream?

## Problem Statement

Recall Module 20, Topic 3's trace of creating a file: updating the inode bitmap, initializing a new inode, updating the containing directory's data, and eventually allocating data blocks — several separate, small, physically scattered writes, even under FFS's clustering. Given Module 17, Topic 3's finding that sequential writes are dramatically faster than scattered ones, could a file system avoid small, scattered writes entirely, converting everything into one long, purely sequential write stream instead?

## Concept

### The Core Idea: Buffer, Then Write Sequentially

> A **log-structured file system (LFS)** buffers a batch of pending writes (inode updates, directory updates, and actual file data) in memory, and periodically flushes the entire batch to disk as one large, purely sequential write — appended to the end of an ever-growing **log** — rather than performing each individual small update as its own separate, scattered disk write.

```
   Traditional (FFS-style) writes for one file update:

   write updated inode          → scattered location A
   write updated directory entry → scattered location B
   write new data block           → scattered location C
   (THREE separate, small, non-adjacent disk writes)

   LFS: buffer all THREE in memory, then flush together:

   [ inode | directory entry | data block ]  ← ONE sequential
                                                  write, appended
                                                  to the end of
                                                  the log
```

This converts what would otherwise be several small, scattered writes (Module 17, Topic 3's slow case) into one large, purely sequential write (Module 17, Topic 3's fast case) — fully embracing the sequential-access advantage, rather than merely mitigating scattering the way FFS's clustering (Topic 1) does.

### The New Problem: Finding the Latest Version of an Inode

Under a traditional file system (Module 20, Topic 1), an inode's location is fixed and directly computable from its inode number (a simple offset calculation into a known inode table). Under LFS, an inode is rewritten to a **new location at the end of the log** every single time it's updated (since nothing is ever overwritten in place — everything is appended) — so a file's inode can be in a *different* physical location every time it changes. How does the file system find the **current, most recent** version of a specific file's inode, if its location keeps changing with every update?

> LFS solves this with an additional structure, often called the **inode map (imap)**: a table, itself also written to the log, that tracks the **current** location of every file's most recent inode. Looking up a file still starts from its inode number, but now requires first consulting the imap to find *where in the log* that inode's current version actually lives, rather than computing a fixed location directly.

### The Cleaning Problem

Because LFS never overwrites data in place — every update is appended as new data at the end of the log — old, now-**stale** versions of inodes and data blocks (superseded by newer writes elsewhere in the log) are left behind, scattered throughout previously-written portions of the log, no longer referenced by anything current. Over time, without intervention, the disk would fill up with these stale, dead versions, even though the *live*, current data might be a small fraction of the total space consumed.

> **Cleaning** (or garbage collection) is the process of periodically scanning older segments of the log, identifying which blocks within them are still live (currently referenced by the imap or current directory structures) versus stale (superseded, dead), and compacting the live blocks together — freeing up the stale space for reuse.

This is LFS's central, unavoidable trade-off: the design that makes writes wonderfully fast and purely sequential also guarantees that space eventually needs active, ongoing reclamation, since nothing is ever overwritten in place.

## Internal Working (Preview)

```
   LFS log, growing over time (append-only):

   [ imap v1 | inode A v1 | data ][ inode A v2 (UPDATED) | data ][ ... ]
                    ▲                        ▲
              now STALE                 CURRENT version —
              (superseded)               imap points HERE now

   Looking up file A's inode:
     1. consult the imap → "file A's current inode is at log position X"
     2. read the inode from position X

   CLEANING (periodically):
     scan an old segment of the log
     for each block: is it still LIVE (referenced by current imap/
                       directory structures) or STALE (superseded)?
     compact the LIVE blocks together elsewhere, freeing the
     entire old segment for reuse
```

## Real-World Analogy

Think of LFS like a diary or journal where you never erase or edit a previous entry — instead, whenever something changes (say, your current address), you simply write a brand-new entry at the very end of the journal stating your new address, leaving the old entry (now outdated) still sitting there on an earlier page. To find your *current* address, you can't just flip to "the address page" directly — you need an index (the imap) telling you which specific, most recent page actually has your current, up-to-date address. Over time, the journal fills up with many outdated, superseded entries — periodically, you'd need to go through, identify which old entries are still actually relevant versus long since superseded, and copy just the still-relevant ones into a fresh, more compact journal, freeing up all the pages that were consumed by entries nobody needs anymore (cleaning).

## Why This Design Is Valued Despite the Cleaning Cost

LFS's write performance advantage can be dramatic, specifically for write-heavy workloads with many small updates — exactly the kind of pattern where FFS's clustering (Topic 1) still incurs meaningful scattered-write overhead, but LFS's "buffer and flush sequentially" approach converts entirely into fast, sequential writes. The cleaning cost is a real, unavoidable consequence of never overwriting in place — but it can be performed in the background, opportunistically, rather than blocking the fast path of ordinary writes, making the trade-off worthwhile for many real-world write-heavy workloads.

## Advantages of Log-Structured File Systems

- **Dramatically faster writes for workloads with many small, frequent updates**, by converting scattered writes into one purely sequential stream.
- **Naturally supports certain crash-recovery advantages** (previewed here, developed further in Module 22) — since writes are appended sequentially, and a well-designed log can be scanned from a known checkpoint forward to recover cleanly after an interruption.

## Disadvantages / Costs

- **The cleaning problem** — stale, superseded data accumulates throughout the log and must be actively identified and reclaimed, a genuinely new kind of maintenance overhead that traditional in-place-update file systems (Module 20, Topic 1; Topic 1's FFS) don't need at all.
- **Finding the current version of any given inode requires an extra indirection step** (the imap), rather than a direct, fixed-offset computation.

## Best Practices

- Recognize LFS as a genuinely different design philosophy from FFS, not merely an incremental refinement — FFS optimizes placement within an in-place-update model; LFS abandons in-place updates entirely in favor of pure sequential appending.
- When evaluating whether an LFS-style design fits a given workload, weigh its write-performance benefits against the ongoing cleaning overhead — write-heavy, small-update-heavy workloads benefit the most; read-heavy, rarely-updated workloads see less relative advantage.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "LFS still updates a file's inode in place, just more efficiently." | LFS never overwrites data in place at all — every update, including inode updates, is appended as new data at the end of the log; an inode's physical location changes with every single update, tracked via the imap. |
| "The cleaning problem in LFS is an occasional edge case, not a fundamental part of the design." | Cleaning is an unavoidable, structural consequence of never overwriting in place — without it, the disk would inevitably fill up with stale, superseded data even while genuinely live data remains a small fraction of total space used; it's a core, ongoing part of any real LFS implementation, not an edge case. |

## Interview Questions

1. **Q: What is the core idea behind a log-structured file system?**
   A: Buffering a batch of pending writes (inode updates, directory updates, file data) in memory and flushing them all together as one large, purely sequential write appended to the end of a log, rather than performing many small, scattered writes as each individual update occurs.

2. **Q: Why does LFS need an inode map (imap), when a traditional file system doesn't need an equivalent structure?**
   A: Because LFS never overwrites data in place — every inode update is appended to a new location in the log, so an inode's physical location changes with every update. The imap tracks each file's current inode location, since it can no longer be computed directly from a fixed offset the way a traditional file system's inode table allows.

3. **Q: What is the cleaning problem in LFS, and why is it unavoidable given the design?**
   A: Since old, superseded versions of inodes and data blocks are never overwritten, they accumulate as stale, dead space throughout the log over time. Cleaning periodically scans old log segments, identifies live versus stale blocks, and compacts the live data to reclaim space — an unavoidable consequence of never updating anything in place.

## Summary

- LFS buffers pending writes in memory and flushes them as one large, sequential write appended to a log, converting what would otherwise be many small, scattered writes into a single fast, sequential one.
- Because inodes are never updated in place, LFS needs an inode map (imap) to track each file's current inode location within the ever-growing log.
- The cleaning problem — reclaiming space from stale, superseded data — is an unavoidable, structural consequence of this append-only design, requiring ongoing background maintenance.
- This closes out the module's coverage of real file system designs built around disk locality — the module summary ties FFS and LFS together before Module 22 covers crash consistency: what happens if a crash occurs mid-update, in either kind of file system.
