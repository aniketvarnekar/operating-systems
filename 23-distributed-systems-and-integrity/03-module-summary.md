# Module 23 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Data Integrity via Checksums** — silent data corruption as a distinct failure mode from RAID and crash consistency, and how checksums combine with redundancy for both detection and correction
- [x] **Networked File Systems: NFS and AFS** — extending the file abstraction across a network, and two genuinely different caching philosophies with distinct network-load/consistency trade-offs

## The Big Picture

This module closed out Persistence (Modules 17–23) by covering two topics that go beyond a single local disk's mechanics: verifying data hasn't silently changed, and extending the file abstraction across a network entirely.

```
   Module 18: RAID — survives an ENTIRE disk failing
   Module 22: journaling — survives a CRASH mid-write
                          │
                          ▼
   Topic 1: checksums — detects SILENT corruption neither
            of the above addresses; combines with RAID's
            redundancy for correction, not just detection
                          │
                          ▼
   Topic 2: NFS vs. AFS — the SAME file abstraction (Module 19),
            now extended across a NETWORK, with a genuine
            network-load vs. consistency trade-off
```

## Practical Connections

- **Why enterprise and cloud storage systems prominently advertise "end-to-end data integrity" or "bit-rot protection" as a feature** — this is Topic 1's checksum-plus-redundancy combination, marketed as a concrete customer-facing guarantee.
- **Why editing a file on a shared network drive can sometimes show you an outdated version until you refresh or reopen it** — this is Topic 2's AFS-style caching trade-off, directly experienced.
- **Why cloud file-sync services (like those used for shared documents) have to make careful, deliberate design decisions about when local changes sync back to a central copy** — the exact same network-load-versus-consistency trade-off from Topic 2, at a larger, internet scale.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| RAID (Module 18) vs. checksums | RAID protects against an entire disk becoming unavailable. Checksums detect silent, per-block corruption on an otherwise healthy disk — a distinct failure mode neither RAID nor journaling addresses alone. |
| Checksum detection vs. correction | A checksum alone only detects that data has changed since it was computed; correcting it requires a separate source of redundancy (like a RAID mirror or parity) to supply the correct replacement value. |
| NFS vs. AFS | NFS keeps clients minimally stateful and relies on the server for nearly every operation (strong consistency, heavier network load). AFS caches whole files locally between open and close (lighter network load, weaker cross-client consistency). |

## What's Next

This module completes Persistence (Modules 17–23) and, with it, all three of this course's big ideas: Virtualization (Modules 02–11), Concurrency (Modules 12–16), and Persistence (Modules 17–23). **Module 24 — Interview Preparation** consolidates everything into ranked, cross-cutting interview questions organized by theme, common problem-style questions, and a final course wrap-up.
