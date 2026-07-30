# Dynamic Relocation (Base and Bounds)

## Learning Objectives

By the end of this section you should be able to:
- Explain what a virtual address is and how it's translated to a physical address under base & bounds
- Explain the role of the base register and the bounds register separately
- Explain why this translation must happen in hardware, not in software, to be practical

## Prerequisites

- Module 06, Topic 1 (The Address Space Abstraction)
- Module 03 (Direct Execution) — the general pattern of hardware-assisted, OS-controlled mechanisms

## Motivation

Module 06 established that every process needs the illusion of a private address space starting at zero. This topic answers the first, most basic version of "how" — a technique historically called dynamic relocation, simple enough to explain in a few sentences, yet foundational enough that its core trick (a small amount of dedicated hardware, configured by the OS, checked on every single memory access) reappears in every more sophisticated mechanism later in this module and Module 08.

## Problem Statement

A process's compiled code refers to memory addresses starting from 0 (Module 06, Topic 1) — for example, an instruction might reference "the data at address 100." But that process's actual data might physically reside anywhere in RAM — say, starting at physical address 32000 — because RAM is shared among many processes (Module 06, Topic 1), and its actual physical location can't be known until the process is actually loaded and run. How does the address "100," as written in the process's own code, actually reach the correct physical location, without the process's code itself needing to know or care where in physical memory it happens to be placed?

## Concept

### Definition

> **Dynamic relocation** (commonly implemented via **base and bounds**) translates every virtual address a process generates into a physical address by adding a fixed offset (the **base**) — while a second value (the **bounds**) is used to verify the virtual address falls within the legitimate range the process is allowed to access at all.

- The **base register** holds the starting physical address where this process's address space actually begins in RAM.
- The **bounds register** (sometimes called the limit register) holds the size of the process's address space, used to check that a given virtual address doesn't exceed what this process is legitimately allowed to use.

### The Translation Formula

```
 physical_address = virtual_address + base

 (permitted only if:  0 <= virtual_address < bounds)
```

If a virtual address fails the bounds check, the hardware raises an exception instead of completing the access — this is precisely how a process is prevented from ever accessing another process's memory: any virtual address it could possibly generate is always checked against its *own* base and bounds, and any attempt to reference memory outside its own legitimate range is caught and blocked before ever reaching physical memory at all.

### Why This Must Happen in Hardware

This translation happens on **every single memory reference** a process makes — every instruction fetch, every data read, every data write. If this addition-and-bounds-check were done in software (by the OS, on every single memory access), the overhead would be catastrophic — effectively turning every memory access into a full OS-mediated operation, destroying the performance goal (Module 06, Topic 1's "efficiency") entirely. Instead, this is implemented directly in the CPU's hardware, in a component called the **Memory Management Unit (MMU)**: the OS configures the base and bounds registers (a privileged operation — Module 03, Topic 2) once, when a process is scheduled to run, and from that point forward, the hardware itself performs the addition and bounds check automatically and near-instantly on every memory access, with zero OS involvement per access.

### Who Sets the Base and Bounds Registers?

Setting these registers is itself a restricted operation (Module 03, Topic 2) — only the OS, running in kernel mode, can update them. The OS sets a process's base and bounds values as part of the same context switch (Module 03, Topic 4) that swaps in that process's saved register state — ensuring the correct translation is always active for whichever process is currently running.

## Internal Working (Preview)

```
   Process's virtual address space         Physical RAM
   (as its own code sees it)               (actual location)

   0   ┌──────────┐                        32000 ┌──────────┐  ← base register = 32000
       │  ...       │   virtual addr 100          │  ...       │
       │  addr 100  │ ───────────────────────────►│ phys 32100 │  (100 + base = 32100)
       │  ...       │                              │  ...       │
  MAX  └──────────┘                        32000+  └──────────┘
       (bounds register = MAX,                      bounds       (any virtual address
        e.g., process size)                                        ≥ MAX is rejected —
                                                                     memory protection)
```

Every process gets its own base and bounds values, loaded fresh at each context switch — so Process A's virtual address 100 and Process B's virtual address 100 translate to entirely different physical addresses, each safely confined to its own region.

## Real-World Analogy

Think of a hotel where every room's internal door numbering starts fresh at "Room 1" from the guest's own perspective, regardless of which actual floor they're staying on. The front desk (the OS) assigns each guest a specific floor offset (the base) when they check in — Room 1 for a guest on floor 3 might actually be physical room 301, while Room 1 for a guest on floor 7 is physical room 701. A small, fast automated elevator-control system (the hardware MMU) applies this offset instantly every time a guest presses a room-number button, and separately checks (the bounds) that the number pressed doesn't exceed how many rooms that guest's floor actually has — refusing the request outright (rather than sending them to a different guest's floor) if it does.

## Why This Design Is Necessary

The alternative — either giving every process direct knowledge of its true physical address (breaking the abstraction from Module 06, Topic 1 and requiring every program to be specially compiled for wherever it happens to load) or performing translation entirely in software (destroying performance) — both fail one of memory virtualization's three goals outright. A small amount of dedicated, fast hardware, configured by a trusted, privileged OS, is what achieves all three goals (transparency, efficiency, protection) simultaneously: the process never needs to know its real physical location (transparency), the translation itself costs almost nothing extra per access (efficiency), and the bounds check prevents any access outside the process's own legitimate region (protection).

## Advantages of Base and Bounds

- **Extremely simple and fast** — just one addition and one comparison per memory access, both trivially implementable in dedicated hardware.
- **Strong isolation** — a process simply cannot generate a virtual address that translates outside its own designated physical region; the hardware itself enforces this on every access.
- **Easy relocation** — an entire process's address space can be moved to a different physical location simply by changing its base register value — the process's own code never needs to change at all.

## Disadvantages of Base and Bounds

- **Internal fragmentation** — recall from Module 06, Topic 1 that a process's address space includes a heap and stack growing toward each other, with free space in between. Under simple base & bounds, that *entire* range (including the currently-unused space between heap and stack) must be allocated as one single, contiguous physical block — wasting real physical memory on space the process isn't actually using yet, simply because there's no way to carve out and separately manage that "gap" region. This specific weakness is exactly what Topic 2 (Segmentation) fixes.
- **Whole-process placement** — the entire address space must fit in one contiguous chunk of physical memory; if physical memory is fragmented into several smaller free chunks, none individually large enough, a process may not be placeable at all even if the *total* free memory would technically be sufficient.

## Best Practices

- When learning any later, more sophisticated translation mechanism (segmentation, paging), explicitly ask "what specific weakness of base & bounds is this fixing?" — nearly every refinement in this module and Module 08 traces back to one of the two disadvantages listed above.
- Keep firmly in mind that the base/bounds *values themselves* are only ever set by the OS (a restricted operation, Module 03, Topic 2) — a user process can generate virtual addresses freely, but can never alter the translation rule applied to them.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The OS translates every memory address in software, checking each one against the process's allowed range." | Doing this in software for every single memory access would be far too slow; the actual translation and bounds check are performed directly by dedicated CPU hardware (the MMU), configured by the OS only once per context switch, not once per memory access. |
| "A process could change its own base or bounds register to access another process's memory." | Setting these registers is a restricted operation (Module 03, Topic 2), executable only by the OS in kernel mode — a user-mode process has no way to modify its own translation parameters. |

## Interview Questions

1. **Q: How does base-and-bounds dynamic relocation translate a virtual address to a physical address?**
   A: It adds the process's base register value to the virtual address to get the physical address, after first checking that the virtual address is within the range specified by the bounds register — an out-of-range access is rejected rather than completed.

2. **Q: Why must this translation happen in hardware rather than in OS software?**
   A: Because it must happen on every single memory access a process makes; doing this in software would impose catastrophic per-access overhead, defeating the efficiency goal of memory virtualization (Module 06, Topic 1). Dedicated hardware (the MMU) performs it near-instantly, with the OS only needing to configure the base/bounds values once per context switch.

3. **Q: What is the main weakness of simple base-and-bounds translation?**
   A: Internal fragmentation — the entire address space (including any currently-unused gap between the heap and stack) must occupy one single, contiguous block of physical memory, wasting physical memory on space the process isn't actually using.

## Summary

- Dynamic relocation translates a virtual address to a physical address by adding a base register value, after checking it against a bounds register — both configured by the OS and enforced by dedicated hardware (the MMU) on every memory access.
- This achieves transparency, efficiency, and protection simultaneously, but at the cost of internal fragmentation: the entire address space, including unused gaps, must fit in one contiguous physical chunk.
- The next topic, Segmentation, generalizes this same base-and-bounds idea to multiple independent segments per process, directly fixing this fragmentation weakness.
