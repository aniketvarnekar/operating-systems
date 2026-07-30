# I/O Devices and the Canonical Protocol

## Learning Objectives

By the end of this section you should be able to:
- Describe the three types of registers a typical device exposes
- Explain the difference between polling and interrupt-driven I/O, and the trade-off between them
- Explain what DMA is and what specific inefficiency it removes

## Prerequisites

- Module 01, Topic 5 (Persistence)
- Module 01, Topic 6 (System Calls and the OS Interface)

## Motivation

Everything from here to the end of the course — files, file systems, RAID, crash consistency — ultimately rests on the OS's ability to talk to physical devices (disks, above all). This topic covers the general pattern every device interaction follows, before Topic 3 narrows in on hard disk drives specifically.

## Problem Statement

A CPU executing instructions and a physical device (a disk, a network card, a keyboard) operate at wildly different speeds and via completely different physical mechanisms. How does the OS issue a command to a device (e.g., "read this block of data"), and how does it know when that command has actually finished — especially since that completion might take a very long time relative to CPU speeds?

## Concept

### The Canonical Device Registers

> Most devices expose (at least) three kinds of registers the OS can read from and write to: a **status register** (readable — reports the device's current state, e.g., busy or ready), a **command register** (writable — tells the device what operation to perform), and a **data register** (readable/writable — used to transfer the actual data being read or written).

A typical interaction: the OS writes data (if needed) to the data register, writes an operation code to the command register (triggering the device to start working), and then needs to find out when that operation completes.

### Polling

> **Polling** is the simplest way to find out when a device operation completes: the OS repeatedly reads the status register in a loop, checking whether the device reports "done," until it does.

This is directly analogous to spinning on a lock (Module 13, Topic 3) — it works, but it wastes CPU cycles the entire time the device is busy, since the CPU has nothing better to do but keep re-checking.

### Interrupt-Driven I/O

> **Interrupt-driven I/O** avoids this waste: instead of polling, the OS issues the command and then lets the CPU go do something else useful entirely (e.g., run a different ready process, Modules 04–05) — the device itself raises a **hardware interrupt** once it completes, which (using the same trap-table mechanism from Module 03, Topic 2) transfers control to a specific kernel handler for that interrupt, informing the OS that the operation is done.

This is a direct, close parallel to Module 10, Topic 2's page fault: both are examples of a hardware-raised trap notifying the kernel of an event, rather than the kernel repeatedly checking for it. The process that requested the I/O is moved to Blocked (Module 02, Topic 2) while waiting, exactly as established back in Module 04, Topic 6 — freeing the CPU to run other ready work in the meantime (Module 04's overlap principle).

### The Trade-off Between Polling and Interrupts

Interrupts aren't strictly, universally better — they have their own real cost (handling an interrupt itself takes some CPU time, similar in kind to a trap's overhead, Module 03, Topic 2). For an operation expected to complete **very quickly**, polling briefly can actually be cheaper than paying an interrupt's fixed handling cost — directly analogous to Module 13, Topic 3's spin-vs-sleep trade-off for locks. Some real device drivers use a hybrid: poll briefly first, then fall back to interrupts if the operation is taking longer than expected.

### DMA: Removing the CPU From Bulk Data Transfer

A separate inefficiency: without special support, the CPU itself would need to copy every single byte of a device's data, one piece at a time, through its own registers — wasting CPU cycles on simple data movement that doesn't need genuine computation at all.

> **Direct Memory Access (DMA)** is a separate, dedicated piece of hardware that can transfer a whole block of data directly between a device and memory, **without the CPU being involved in each individual byte's transfer at all** — the CPU simply tells the DMA controller the source, destination, and size of the transfer, and is then free to do other work while the DMA controller handles the bulk data movement independently, raising an interrupt only once the *entire* transfer is complete.

## Internal Working (Preview)

```
   POLLING:                             INTERRUPT-DRIVEN + DMA:

   OS: write command                    OS: write command, tell DMA
   loop {                                    controller src/dest/size
     read status register        vs.    OS: goes and does OTHER WORK
     if "done", break                        (this process → Blocked,
   }                                          Module 02, Topic 2)
   (CPU busy-waits the                  DMA controller: moves the
    ENTIRE time — wasteful)                  data independently
                                          Device/DMA: raises an INTERRUPT
                                              once the WHOLE transfer
                                              completes
                                          OS: interrupt handler wakes
                                              the waiting process
```

## Real-World Analogy

Polling is like repeatedly calling a restaurant every thirty seconds to ask "is my order ready yet?" — functional, but you're spending your own time doing nothing but asking. Interrupt-driven I/O is like the restaurant calling *you* the moment your order is ready, letting you go do something else entirely in the meantime. DMA is like having a dedicated delivery driver handle the entire trip from restaurant to your door — you (the CPU) don't personally carry each individual food item one piece at a time; you're just notified once the driver has completed the whole delivery.

## Why This Design Is Necessary

Without interrupts, an OS could either poll (wasting CPU cycles) or, worse, do nothing at all until an operation completed — unacceptable given how much slower physical devices are than the CPU (Module 01, Topic 5's persistence discussion). Without DMA, the CPU itself would be tied up moving every byte of every device transfer, a poor use of a resource capable of far more complex work in the meantime. Both mechanisms exist specifically to keep the CPU free for other useful work (Module 04's overlap principle) while slow, physical operations happen largely independently in the background.

## Advantages of Interrupt-Driven I/O with DMA

- **Frees the CPU for other useful work** during both the wait for device completion and the bulk data transfer itself, directly extending Module 04, Topic 6's overlap principle to device I/O generally.
- **Scales well** — the CPU's involvement per I/O operation is minimal (issue the command, handle one interrupt at completion), regardless of how much data is actually being transferred.

## Disadvantages / Costs

- **Interrupts have real, non-zero handling overhead** — for very fast operations, polling briefly can sometimes be cheaper, exactly paralleling Module 13, Topic 3's lock spin-vs-sleep trade-off.
- **DMA requires dedicated hardware support** and adds real complexity to correctly coordinating memory access between the CPU and the DMA controller (e.g., ensuring the CPU doesn't read a buffer the DMA controller hasn't finished writing to yet).

## Best Practices

- When explaining why a slow disk operation doesn't freeze the whole system, connect it directly to this topic: the OS blocks only the specific requesting process (Module 02, Topic 2) and lets the scheduler (Modules 04–05) run other ready work, informed by the eventual interrupt rather than by polling.
- Recognize the recurring pattern across this course: traps (Module 03), page faults (Module 10), and device interrupts (this topic) are all the same underlying mechanism — a hardware-forced, safe transfer of control to a fixed kernel handler — applied to different triggering events.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Interrupts are always strictly better than polling." | For very fast operations, an interrupt's own handling overhead can exceed the cost of simply polling briefly — the better choice depends on expected operation duration, exactly like the spin-vs-sleep trade-off for locks (Module 13, Topic 3). |
| "DMA means the device transfers data without the OS's involvement at all." | The OS still initiates the transfer (telling the DMA controller the source, destination, and size) and still handles the completion interrupt — DMA specifically removes the CPU from needing to move each individual byte itself, not from the process entirely. |

## Interview Questions

1. **Q: What's the difference between polling and interrupt-driven I/O?**
   A: Polling has the OS repeatedly check a device's status register in a loop until it reports completion, wasting CPU cycles the entire time. Interrupt-driven I/O lets the CPU do other work while waiting, with the device raising a hardware interrupt (via the same trap mechanism as Module 03) to notify the kernel once the operation completes.

2. **Q: What is DMA, and what specific inefficiency does it remove?**
   A: Direct Memory Access is dedicated hardware that transfers data directly between a device and memory without the CPU handling each individual byte. It removes the need for the CPU to spend its own cycles copying bulk data, freeing it for other work while the transfer happens independently.

3. **Q: When might polling actually be preferable to using an interrupt?**
   A: When the expected operation duration is very short — an interrupt's own handling overhead can exceed the cost of briefly polling, directly analogous to the spin-vs-sleep trade-off for locks.

## Summary

- Devices expose status, command, and data registers as the canonical interface the OS interacts with.
- Polling wastes CPU cycles checking for completion; interrupt-driven I/O lets the CPU do other work and is notified via a hardware interrupt, the same underlying trap mechanism used by system calls and page faults.
- DMA removes the CPU from needing to handle bulk data transfer byte-by-byte, freeing it for other work during the transfer itself.
- The next topic covers how the OS hides all of this device-specific detail behind a clean, uniform interface: the device driver.
