# Module 22 — Crash Consistency

## Module Goal

By the end of this module, you will understand **the crash consistency problem** — precisely why a multi-step file system update (Module 20, Topic 3's traces) can leave the file system internally inconsistent if a crash occurs mid-update — and the two major solution strategies: the traditional after-the-fact `fsck` approach, and journaling, which prevents inconsistency from ever occurring in the first place.

## Topics Covered in This Module

1. **[The Crash Consistency Problem and fsck](01-the-crash-consistency-problem-and-fsck.md)** — Exactly what can go wrong when a crash interrupts a multi-step update, and the traditional full-disk-scan repair approach.
2. **[Journaling](02-journaling.md)** — Write-ahead logging: recording intent before acting, so any crash can be safely completed or safely undone.
3. **[Module Summary](03-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 20, Topic 3 (Walking Through File Operations) — this module directly answers "what if a crash happens mid-trace?"
- Module 01, Topic 5 (Persistence) — the crash-mid-write scenario introduced conceptually there is fully resolved here.

## How to Study This Module

Read in order. Topic 1 makes the danger completely concrete, using Module 20, Topic 3's own file-creation trace as the worked example — showing exactly which specific inconsistencies can result from a crash between specific steps, and the traditional (but slow, and imperfect) fsck approach to detecting and repairing them after the fact. Topic 2 covers the better modern answer: journaling, which changes the *order and manner* of writes so that a crash at literally any point still leaves a safely recoverable state — directly fulfilling the promise Module 01, Topic 5 made at the very start of this course.
