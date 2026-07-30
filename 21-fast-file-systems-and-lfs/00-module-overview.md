# Module 21 — Fast File Systems and LFS

## Module Goal

By the end of this module, you will understand **two real-world file system designs built specifically around Module 17, Topic 3's central finding — sequential disk access is dramatically faster than random access**: FFS, which places related data physically close together, and log-structured file systems (LFS), which convert essentially all writes into purely sequential ones.

## Topics Covered in This Module

1. **[FFS and the Locality Principle](01-ffs-and-the-locality-principle.md)** — How the Fast File System groups related files and directories physically close together on disk to minimize seek time.
2. **[Log-Structured File Systems](02-log-structured-file-systems.md)** — Buffering writes in memory and flushing them sequentially as one large, append-only log, plus the cleaning problem this introduces.
3. **[Module Summary](03-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 17, Topic 3 (Hard Disk Drive Mechanics) — this entire module is a direct engineering response to that topic's findings.
- Module 20 in full — both designs still use inodes, bitmaps, and directory entries; this module changes how those structures are physically arranged and written, not what they fundamentally are.

## How to Study This Module

Read in order. Topic 1 shows the most direct possible response to Module 17, Topic 3: since seek time dominates, physically cluster data likely to be accessed together, minimizing how far the disk arm has to travel. Topic 2 takes a more radical approach: instead of just clustering related data, convert the entire write pattern into one purely sequential stream — a design that fully embraces Module 17, Topic 3's sequential-access advantage, at the cost of a genuinely new problem (space reclamation) that Topic 2 covers directly.
