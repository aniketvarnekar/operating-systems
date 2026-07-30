# Module 21 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **FFS and the Locality Principle** — cylinder groups, clustering related data, and the large-file trade-off
- [x] **Log-Structured File Systems** — buffering and sequential appending, the imap, and the cleaning problem

## The Big Picture

This module presented two genuinely different, real-world engineering responses to the same physical reality established back in Module 17, Topic 3: sequential disk access dramatically outperforms random access. FFS mitigates the problem by clustering related data; LFS eliminates the problem for writes almost entirely by converting them into pure sequential appends — at the cost of a new, unavoidable maintenance burden.

```
   Module 17, Topic 3: sequential access >>> random access (mechanically)
                          │
           ┌──────────────┴──────────────┐
           ▼                              ▼
   Topic 1: FFS                    Topic 2: LFS
   Keep an IN-PLACE-UPDATE          Abandon in-place updates —
   model, but CLUSTER related        buffer writes, flush as ONE
   data (cylinder groups) to         sequential append to a LOG
   minimize seek distance                    │
                                              ▼
                                     needs an imap (inode location
                                     changes on every update) +
                                     CLEANING (reclaim stale space)
```

## Practical Connections

- **Why file systems historically performed noticeably better when kept below a certain "percent full" threshold** — this connects to Topic 1's discussion of cylinder groups losing placement flexibility as they fill, and Topic 2's cleaning needing free space to work with.
- **Why some modern storage systems (particularly those built for SSDs and write-heavy workloads, like certain key-value stores and even some SSD firmware itself) use log-structured, append-only designs** — Topic 2's core idea (never overwrite in place, clean later) generalizes well beyond traditional spinning-disk file systems.
- **Why "defragmentation" was historically a meaningful maintenance operation on some file systems, but matters far less on modern SSD-based systems** — Topic 1's entire premise (physical seek distance matters) is specific to mechanical disks; SSDs' different physical characteristics (Module 17, Topic 3) change this calculus substantially.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| FFS vs. LFS | FFS keeps an in-place-update model but clusters related data physically close together to reduce seek time. LFS abandons in-place updates entirely, converting all writes into one sequential, append-only log. |
| Cylinder group vs. the whole disk | A cylinder group is a self-contained sub-region with its own local inodes, bitmaps, and data — not the entire disk's single, disk-wide pool of each, which is what a naive layout (Module 20) would use. |
| Cleaning (LFS) vs. ordinary free-space reclamation (Module 20) | Ordinary reclamation (Module 20, Topic 2) simply marks blocks free once a reference count hits zero. LFS cleaning must actively scan old log segments to distinguish live from stale data and compact the live portions, since nothing is overwritten in place. |

## What's Next

Both FFS and LFS involve multi-step updates across several on-disk structures — and neither design, as described so far, has addressed what happens if a crash occurs partway through one of those updates. **Module 22 — Crash Consistency** covers exactly this: the crash-consistency problem in full, the traditional fsck approach to detecting and repairing inconsistencies after the fact, and journaling — the write-ahead logging technique that prevents inconsistency from occurring in the first place.
