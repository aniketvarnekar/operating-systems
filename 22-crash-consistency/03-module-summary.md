# Module 22 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Crash Consistency Problem and fsck** — a concrete orphaned-inode example, and fsck's full-disk-scan detection/repair approach
- [x] **Journaling** — write-ahead logging's journal/commit/checkout sequence, and why recovery only needs the journal, not the whole disk

## The Big Picture

This module finally resolved the question Module 01, Topic 5 raised at the very beginning of the course: what happens if a crash occurs mid-write? Module 20, Topic 3's concrete multi-step trace made the danger precise; this module covered both the traditional after-the-fact answer (fsck) and the modern, structurally superior answer (journaling).

```
   Module 01, Topic 5: "what if a crash happens mid-write?" (posed)
   Module 20, Topic 3: a concrete multi-step trace (the danger, made real)
                          │
           ┌──────────────┴──────────────┐
           ▼                              ▼
   Topic 1: fsck                   Topic 2: journaling
   Detect + repair AFTER the fact   PREVENT inconsistency by recording
   Must scan the ENTIRE disk        intent FIRST, in a small journal
   Cannot recover lost intent       Recovery only reads the journal —
                                     fast, precise, always correct
```

## Practical Connections

- **Why modern operating systems recover from an improper shutdown in seconds, while older systems could take minutes on a large disk** — this is journaling's fast, journal-only recovery versus fsck's full-disk-scan requirement, made concrete.
- **Why database systems use their own write-ahead logs (WALs), conceptually identical to file system journaling** — the exact same "record intent before acting" pattern, applied at the database transaction level instead of the file system block level.
- **Why "lost+found" directories exist on some UNIX-like systems** — this is a direct, visible artifact of fsck's repair process (Topic 1): orphaned inodes it finds are placed there rather than being silently discarded.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| fsck vs. journaling | fsck detects and repairs inconsistencies after the fact by scanning the entire disk. Journaling prevents inconsistency by recording intent in a small journal before acting, letting recovery replay or discard journal entries without a full-disk scan. |
| An orphaned inode vs. a lost block | An orphaned inode is marked in-use with valid metadata but referenced by no directory entry. A lost block is marked in-use in the data bitmap but referenced by no inode — a related but distinct inconsistency pattern. |
| Journal write vs. commit vs. checkout | The journal write records intended changes; the commit marks that recording as complete and safe to act on; the checkout actually applies the changes to the real, permanent structures — the commit is the critical dividing line for crash recovery. |

## What's Next

Modules 17–22 covered persistence on a single machine: devices, disks, RAID, file/directory abstractions, on-disk implementation, locality-aware design, and crash consistency. **Module 23 — Distributed Systems and Integrity** extends Persistence in two directions: data integrity via checksums (detecting corruption that crash consistency alone doesn't address), and an introduction to networked file systems (NFS and AFS) — what changes once "the disk" is no longer physically local to the machine using it.
