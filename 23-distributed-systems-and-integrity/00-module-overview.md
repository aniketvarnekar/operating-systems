# Module 23 — Distributed Systems and Integrity

## Module Goal

By the end of this module, you will understand **data integrity via checksums** — detecting corruption that crash consistency alone doesn't address — and get an introduction to **networked file systems**, covering what changes once "the disk" a program accesses is no longer physically local to the machine using it.

## Topics Covered in This Module

1. **[Data Integrity via Checksums](01-data-integrity-via-checksums.md)** — Detecting silent data corruption that neither RAID nor journaling protects against.
2. **[Networked File Systems: NFS and AFS](02-networked-file-systems-nfs-and-afs.md)** — Accessing files over a network transparently, and two different design philosophies for client-side caching.
3. **[Module Summary](03-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 18 (RAID) — checksums address a different failure mode than RAID's disk-failure tolerance.
- Module 22 (Crash Consistency) — checksums address a different failure mode than crash-mid-write inconsistency.
- Module 19 (Files and Directories) — networked file systems extend that same abstraction across a network.

## How to Study This Module

Read in order. Topic 1 introduces a failure mode neither RAID (Module 18) nor journaling (Module 22) protects against at all: silent corruption of data that was never in the middle of being written or experiencing an outright disk failure. Topic 2 then extends the entire Persistence story across a network — the same file abstraction (Module 19), but now with a genuinely new set of concerns (network failures, caching, consistency across multiple clients) that a purely local file system never had to consider.
