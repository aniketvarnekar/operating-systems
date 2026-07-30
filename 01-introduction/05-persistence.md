# Persistence

## Learning Objectives

By the end of this section you should be able to:
- Define persistence and explain why memory alone can never provide it
- Explain, with a concrete example, why "just write the bytes to disk" is not sufficient to survive a crash
- Identify which later modules are direct deep dives into persistence

## Prerequisites

- Topic 3 (Virtualization) — persistence is best understood as the third leg alongside virtualization and concurrency, not a prerequisite of either.

## Motivation

Virtualization (Topic 3) and concurrency (Topic 4) both concern things that happen *while a program is running*. Persistence concerns something different and, in some ways, harder: making data outlive the program entirely — surviving the process exiting, the machine rebooting, or even the power failing at the worst possible instant. Modules 17–23 are all deep dives into this single idea.

## Problem Statement

Everything a running program keeps in memory (its address space — Module 06) is **volatile**: the instant the process ends, or the machine loses power, that memory's contents are gone forever. Yet you clearly expect a document you saved yesterday to still exist today, even after you shut your laptop down overnight. Something has to make certain data outlive the process — and the machine's power state — that created it.

A harder version of the same problem: what happens if the machine loses power **in the middle** of saving a file — after some, but not all, of the new data has been written to disk? Naively, you might expect this to just leave you with a partially-written (but at least still valid) old-or-new file. In reality, without careful design, an interrupted write can leave a file system's own internal bookkeeping structures in an inconsistent state — potentially corrupting data that wasn't even being touched by the interrupted write, or losing track of which disk blocks belong to which file entirely.

## Concept

### Definition

> **Persistence** is the set of techniques and guarantees an operating system provides to ensure that data written to durable storage (a disk) remains correctly readable later — across process termination, reboots, and even a crash or power loss occurring mid-write.

### Why Persistence Needs Its Own Hardware and Its Own Rules

Physical RAM is fast but volatile (loses its contents without continuous power). A hard disk drive or solid-state drive is slower but **non-volatile** — its contents survive with no power at all. This single hardware distinction is why persistence is a genuinely separate problem from virtualization: virtual memory (Modules 06–11) is about managing a *volatile* resource efficiently; persistence is about safely committing data to a *non-volatile* one, including surviving interruptions mid-write.

### The Three Layers of the Persistence Story

1. **The hardware layer** (Module 17): how I/O devices and disks physically work — why a hard disk drive's spinning mechanics make certain access patterns dramatically faster than others, and how RAID (Module 18) combines multiple disks for either more capacity, more speed, or more safety against a single disk failing.
2. **The abstraction layer** (Modules 19–21): the file and directory abstractions programs actually use, and how a file system organizes raw disk blocks into named, structured files underneath that abstraction — including designs (like FFS and log-structured file systems) that specifically exploit how disks physically behave for better performance.
3. **The crash-consistency layer** (Module 22): the specific techniques (like journaling) that guarantee a file system's own bookkeeping stays internally consistent even if the power is cut at the worst conceivable moment mid-update — directly solving the "interrupted write" problem above.

## Internal Working (Preview)

```
 Process running  ──uses──►  Volatile Memory (RAM)
      │                       (address space, Module 06 —
      │                        gone the instant power is lost)
      │
      └──explicitly saves──►  Non-Volatile Storage (Disk)
                               (files, Module 19 —
                                survives power loss,
                                *if* written carefully — Module 22)
```

The crucial detail: moving data from the top box to the bottom box is not instantaneous or automatically safe — a crash during that move is exactly the scenario crash-consistency techniques (Module 22) are built to survive.

## Real-World Analogy

Think of writing a check to update a company's official ledger. If the ledger-keeper is interrupted mid-entry — say, they've written the new balance but haven't yet crossed out the old one, or vice versa — a well-run accounting system uses a technique like keeping a running journal of intended changes *before* editing the master ledger itself: "I am about to change entry #42 from $100 to $150," written down first, separately. If the ledger-keeper is interrupted right after writing that journal note but before editing the master ledger, anyone can later read the journal and either finish or safely redo the edit — the master ledger is never left in a confusing, half-updated, uninterpretable state. This journal-first technique is precisely the real-world equivalent of **journaling** (Module 22), one of the core solutions to crash consistency.

## Why Persistence Requires Deliberate Design

If an OS simply wrote data to disk in the most straightforward possible way (update bookkeeping structures and file contents in whatever order is convenient), a crash at the wrong instant could leave the file system's own internal structures in a state that doesn't correspond to any valid before-or-after snapshot — data could become unreadable, or worse, silently corrupted in a way that looks valid but isn't. Real file systems deliberately order and/or duplicate their writes (via journaling or similar techniques, Module 22) specifically so that *any* point at which a crash occurs still leaves the disk in a state the system can recover to something consistent.

## Advantages of Careful Persistence Design

- **Durability** — data genuinely survives process crashes, reboots, and power loss, not just "normal" shutdowns.
- **Consistency** — the file system's own internal structures remain trustworthy and readable even after an interruption mid-write, instead of becoming silently corrupted.

## Disadvantages / Trade-offs

- **Performance cost** — techniques like journaling mean some data is effectively written twice (once to a journal, once to its final location), which costs real time and disk bandwidth in exchange for crash safety.
- **Complexity** — correctly reasoning about "what if a crash happens at literally any instruction boundary during this update" is one of the hardest correctness problems in systems software, which is why Module 22 exists as its own dedicated module.

## Best Practices

- Never assume "the write call returned successfully" means data is safely, durably on disk in a crash-proof way — different levels of guarantee exist (buffered in the OS, physically on the disk platter, etc.), and Module 17 and Module 22 both address this in depth.
- When thinking about any persistence mechanism, always ask "what does this guarantee if a crash happens at the worst possible moment, mid-operation?" — that question is the entire point of crash consistency.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Once data is in a file, it's automatically safe from any kind of crash." | Only if the file system used a design that specifically accounts for crash consistency (Module 22); a naive write order can leave a file system's own metadata corrupted by an ill-timed crash. |
| "Persistence is just 'files exist on disk instead of memory' — nothing more subtle than that." | The genuinely hard part isn't that disks are non-volatile (that's just a hardware fact) — it's guaranteeing correctness despite an interruption occurring at literally any point during a multi-step update. |

## Interview Questions

1. **Q: What is persistence, in the OS sense?**
   A: The set of techniques ensuring data written to non-volatile storage remains correctly readable later, including after a process ends, a reboot, or a crash/power loss occurring mid-write.

2. **Q: Why isn't "just write it to disk" automatically enough to guarantee data survives a crash correctly?**
   A: Because a crash can occur mid-write, partway through updating a file system's own internal bookkeeping structures — without deliberate techniques like journaling, this can leave those structures in an inconsistent state, not just an incomplete file.

3. **Q: Why is persistence considered a separate concern from virtualization, even though both involve memory-like resources?**
   A: Virtualization (Modules 06–11) manages a volatile resource (RAM) efficiently while a program runs; persistence (Modules 17–23) is about safely committing data to non-volatile storage such that it survives the program — and the machine's power — going away entirely, including surviving an interruption mid-write.

## Summary

- Persistence ensures data survives beyond a running process — including reboots and power loss mid-write.
- The hard part isn't disks being non-volatile (a hardware fact); it's guaranteeing correctness even if a crash occurs at the worst possible instant during an update.
- Modules 17–23 build up the full story: device/disk mechanics, file and file-system abstractions, and crash-consistency techniques like journaling.
- Together, Virtualization, Concurrency, and Persistence (Topics 3–5) are the three big ideas this entire course is organized around.
