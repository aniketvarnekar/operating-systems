# Module 17 — I/O Devices and Disks

## Module Goal

By the end of this module, you will understand **how the OS communicates with physical I/O devices in general, and hard disk drives in particular** — the beginning of Persistence, this course's third and final major theme (introduced conceptually in Module 01, Topic 5). This module lays the hardware foundation that Modules 18–23 build the entire file-system and crash-consistency story on top of.

## Topics Covered in This Module

1. **[I/O Devices and the Canonical Protocol](01-io-devices-and-the-canonical-protocol.md)** — The general model every device interaction follows: status/command/data registers, polling vs. interrupts, and DMA.
2. **[Device Drivers](02-device-drivers.md)** — How the OS hides device-specific details behind a uniform interface.
3. **[Hard Disk Drive Mechanics](03-hard-disk-drive-mechanics.md)** — Platters, tracks, sectors, and why physical geometry makes some access patterns dramatically faster than others.
4. **[Disk Scheduling](04-disk-scheduling.md)** — SSTF, SCAN, and C-LOOK: policies for ordering pending disk requests to exploit that geometry.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 01, Topic 5 (Persistence)
- Module 01, Topic 6 (System Calls) — devices are accessed through the same trap-based mechanism.
- Module 04 (CPU Scheduling) — helpful context, since disk scheduling (Topic 4) directly parallels CPU scheduling's mechanism/policy split.

## How to Study This Module

Read in order. Topic 1 gives you the general vocabulary every device (not just disks) shares. Topic 2 shows how the OS hides that device-specific detail behind a clean, uniform interface — directly echoing Module 06, Topic 1's transparency goal, now applied to hardware devices instead of memory. Topics 3–4 narrow in on hard disk drives specifically: the physical mechanics first (why geometry matters at all), then the scheduling policies that exploit that geometry — a direct, deliberate parallel to Module 04's CPU scheduling mechanism/policy split.
