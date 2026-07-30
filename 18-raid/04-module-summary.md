# Module 18 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Why RAID? Striping and RAID 0** — combining disks for capacity and parallel performance, with zero (indeed negative) reliability benefit
- [x] **Mirroring: RAID 1** — full duplication for straightforward fault tolerance, at a steep 50% capacity cost
- [x] **Parity-Based RAID: RAID 4 and RAID 5** — a compact, mathematical redundancy scheme, and the write-bottleneck fix that distinguishes RAID 5 from RAID 4

## The Big Picture

This module walked through RAID's core trade-off triangle — capacity, performance, reliability — showing how each level sits at a different point in that space, in order of increasing sophistication.

```
                        CAPACITY
                          ▲
                          │  RAID 0 (full capacity,
                          │   zero reliability)
                          │
   RAID 4/5 ──────────────┼────────────── (near-full capacity,
   (N-1)/N usable,         │                single-disk fault tolerance)
   single-disk tolerant     │
                          │
                          │  RAID 1 (half capacity,
                          │   simple, robust fault tolerance)
                          ▼
                     RELIABILITY

   Performance: RAID 0 fastest (no redundancy overhead at all);
                RAID 1 fast reads, no write benefit;
                RAID 4 write-bottlenecked; RAID 5 fixes that bottleneck
```

## Practical Connections

- **Why a "scratch" or temporary compute volume might deliberately use RAID 0, while a database's critical data volume uses RAID 1 or RAID 5** — this is Topic 1–3's trade-off triangle, applied as an explicit, deliberate infrastructure decision matched to each workload's actual reliability needs.
- **Why replacing a failed disk in a RAID 5 array triggers a long, I/O-intensive "rebuild" process, during which performance is often noticeably degraded** — this is Topic 3's reconstruction cost made concrete: every remaining disk must be read in full to recompute the missing disk's contents via parity.
- **Why larger RAID 5/6 arrays are generally more capacity-efficient than smaller ones** — Topic 3's (N−1)/N formula directly explains why: the one-disk parity overhead becomes a smaller fraction of the total as N grows.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| RAID 0 vs. RAID 1 | RAID 0 stripes with zero redundancy — full capacity, but worse reliability than a single disk. RAID 1 mirrors fully — half capacity, but survives any single disk's failure. |
| Mirroring vs. parity | Mirroring keeps a full, duplicate copy (100% overhead per protected disk). Parity computes a single reconstructable value from multiple disks (one disk's overhead total, regardless of array size) — far more capacity-efficient for larger arrays. |
| RAID 4 vs. RAID 5 | RAID 4 dedicates one fixed disk to parity, creating a write bottleneck. RAID 5 rotates parity across all disks, distributing writes and relieving that bottleneck. |

## What's Next

Modules 17–18 built the physical and array-level storage foundation: device protocols, disk mechanics and scheduling, and RAID's capacity/performance/reliability trade-offs. **Module 19 — Files and Directories** moves up a layer, to the abstractions programs actually use on top of this physical storage — the file and directory abstractions, hard versus soft links, and mounting — before Module 20 covers how a file system actually implements these abstractions on disk.
