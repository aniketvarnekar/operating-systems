# Module 17 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **I/O Devices and the Canonical Protocol** — status/command/data registers, polling vs. interrupts, and DMA
- [x] **Device Drivers** — concentrating device-specific knowledge behind a uniform interface, extending Module 06's transparency goal to hardware
- [x] **Hard Disk Drive Mechanics** — platters, tracks, sectors, and the seek time/rotational latency/transfer time breakdown
- [x] **Disk Scheduling** — SSTF, SCAN, and C-LOOK, directly paralleling Module 04's CPU scheduling trade-offs

## The Big Picture

This module began Persistence by building from the ground up: the general device protocol (Topic 1), the abstraction layer that hides device-specific detail (Topic 2), the specific physical mechanics of hard disks (Topic 3), and the scheduling policies that exploit those mechanics (Topic 4) — a foundation every later Persistence module (18–23) builds directly on top of.

```
   Module 01, Topic 5: Persistence (conceptual introduction)
                          │
                          ▼
   Topic 1: the canonical device protocol (registers, polling/interrupts, DMA)
                          │
                          ▼
   Topic 2: device drivers — hide device specifics behind a uniform interface
                          │
                          ▼
   Topic 3: HDD mechanics — WHY seek time and rotation make some
            access patterns dramatically slower than others
                          │
                          ▼
   Topic 4: disk scheduling — ORDERING requests to exploit Topic 3's
            physical realities (a direct parallel to Module 04's
            CPU scheduling mechanism/policy split)
```

## Practical Connections

- **Why "eject before removing a USB drive" matters, and why an OS notification confirms it's safe** — this connects to Topic 1's interrupt-driven completion model: the OS needs confirmation that pending writes have actually finished before physical removal is safe.
- **Why database and file system documentation frequently recommends "sequential write patterns" for best performance on spinning disks** — this is Topic 3's core finding, turned into practical, actionable advice.
- **Why cloud storage tiers offer both fast "SSD-backed" and slower, cheaper "HDD-backed" options** — Topic 3's mechanical cost model is precisely why HDDs remain cheaper per byte but slower for random access, a real trade-off customers explicitly choose between.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Polling vs. interrupt-driven I/O | Polling repeatedly checks a status register in a loop, wasting CPU cycles. Interrupt-driven I/O lets the CPU do other work and is notified via a hardware interrupt once the operation completes. |
| Seek time vs. rotational latency | Seek time is the disk arm physically moving to the correct track. Rotational latency is waiting for the platter's spin to bring the correct sector under the (already-positioned) head — two distinct mechanical delays. |
| SSTF vs. SCAN/C-LOOK | SSTF always services the closest pending request, risking starvation for distant ones. SCAN/C-LOOK sweep the disk in one direction, guaranteeing every request is serviced within a bounded sweep, at a small cost to per-decision optimality. |

## What's Next

This module covered a single disk's mechanics and scheduling. **Module 18 — RAID** covers what happens when multiple disks are combined together — for greater capacity, greater performance (via parallelism across disks), or greater reliability (surviving an individual disk's failure) — and the specific trade-offs between RAID levels 0, 1, 4, and 5.
