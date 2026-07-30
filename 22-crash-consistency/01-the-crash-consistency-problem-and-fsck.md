# The Crash Consistency Problem and fsck

## Learning Objectives

By the end of this section you should be able to:
- Explain precisely what the crash consistency problem is, using a concrete multi-step update as an example
- Identify the specific inconsistent states that can result from a crash at different points in that update
- Explain what fsck does, and its two major limitations

## Prerequisites

- Module 20, Topic 3 (Walking Through File Operations)
- Module 01, Topic 5 (Persistence)

## Motivation

Module 01, Topic 5 raised this problem in the abstract, right at the start of the course: what happens if the machine loses power mid-write? Module 20, Topic 3 then gave you a concrete, multi-step trace of what a real file system operation (like file creation) actually involves. This topic connects the two directly: exactly what goes wrong if a crash interrupts that specific multi-step trace, and the traditional way file systems have dealt with it.

## Problem Statement

Recall Module 20, Topic 3's file-creation trace: (1) find a free inode via the inode bitmap, (2) initialize that inode, (3) add a directory entry referencing it, and (4) eventually allocate data blocks. Each of these is a **separate** write to disk. If the machine loses power (or the OS crashes) after step 2 completes but **before** step 3 does, what state is the file system left in when it reboots?

## Concept

### A Concrete Inconsistency

If a crash occurs exactly between Module 20, Topic 3's steps 2 and 3:

- The **inode bitmap** has been updated to mark the new inode number as in-use (step 1 completed).
- The **inode itself** has been initialized with valid-looking metadata (step 2 completed).
- But **no directory entry** anywhere references this inode number at all (step 3 never happened).

> The result: an inode that is marked "in use" and contains real metadata, but is **unreachable** from any directory — a real inconsistency between the bitmap (which says "in use") and the actual directory structure (which has no path leading to it at all). This specific pattern is sometimes called an **orphaned inode**.

Other crash points in other operations produce other, different specific inconsistencies — for example, a crash between allocating a data block (updating the data bitmap) and updating the owning inode's block pointers to reference it could leave a block marked "in use" in the bitmap while no inode actually points to it (a "lost" block, wasting space but not otherwise dangerous), or, in a different ordering, an inode could end up pointing to a block that the bitmap still shows as "free" (a much more dangerous case, since that block could later be allocated to a *different* file, corrupting both).

### Why This Class of Problem Is Genuinely Hard

The fundamental issue: a logically single "operation" (create a file; write a data block) actually requires **multiple separate, physically distinct disk writes**, and a crash can occur at **any** point between them — after some have completed, before others have even started. There is no way to make several genuinely separate disk writes complete as a single, indivisible unit purely through the writes themselves; some additional mechanism is needed to handle every possible interruption point safely.

### fsck: Detect and Repair After the Fact

> **fsck** (file system check) is a traditional approach: after a crash, before the file system is used again, a dedicated program scans the **entire** disk, checking every inode, bitmap, and directory structure for consistency, and attempts to repair whatever inconsistencies it finds (e.g., an orphaned inode might be moved into a special "lost+found" directory, marked recoverable but no longer silently invisible).

## Internal Working (Preview)

```
   File creation trace (Module 20, Topic 3):

   Step 1: inode bitmap → mark inode #500 IN USE
   Step 2: inode #500 → initialize metadata
                    │
              CRASH HERE
                    │
   Step 3: directory → add entry ("newfile.txt", 500)   ← NEVER HAPPENS
   Step 4: (later) allocate data blocks

   RESULT after reboot:
     inode bitmap: says #500 is IN USE
     inode #500:    has valid-looking metadata
     directory:      NO entry anywhere points to #500
     → ORPHANED INODE — inconsistent, wastes space, invisible
       to normal path-based access


   fsck's approach:
     scan EVERY inode, EVERY bitmap, EVERY directory on the ENTIRE disk
     cross-check them all against each other for consistency
     repair what it finds (e.g., move orphaned inodes to lost+found)
```

## Real-World Analogy

Think of the crash-mid-update scenario like a librarian who has just finished writing up a brand-new catalog card for a book (initializing the inode) — recording its title, author, and a fresh catalog number they've just claimed as unused (the bitmap update) — but is interrupted (a power outage) before they get around to actually adding that catalog number to the shelf's own index of "which books live here" (the directory entry). The book itself, if it even physically exists yet, is now essentially lost to the system: the catalog number is marked as claimed, a card exists for it, but no shelf's index will ever lead a patron to look for it. fsck is like a library-wide audit team that, after any such disruption, walks through literally every catalog card, every shelf index, and every claimed catalog number in the *entire* library, cross-checking all of them against each other to find and fix exactly this kind of orphaned, unreferenced entry.

## Why fsck Has Serious Limitations

- **It must scan the entire disk**, regardless of how small the actual interrupted operation was — checking every inode and every directory on a large, modern disk can take an extremely long time, becoming increasingly impractical as disk capacities have grown dramatically over the years.
- **It can only detect and repair *structural* inconsistencies** (like an orphaned inode) — it generally **cannot** recover the actual, intended semantic content of an interrupted operation (e.g., it can't know what the interrupted write was actually trying to accomplish, only that something looks structurally wrong); its repairs are aimed at restoring internal consistency, not necessarily preserving the user's actual lost work.

## Advantages of the fsck Approach

- **Conceptually simple** — no special-case logic is needed during normal, uninterrupted operation; the complexity is entirely concentrated in the after-the-fact repair tool.
- **Works as a general safety net** regardless of exactly which specific operation was interrupted or how.

## Disadvantages of the fsck Approach

- **Full-disk scan cost scales with disk size**, not with how much data was actually affected by the crash — a severe, growing practical problem as disks have gotten dramatically larger.
- **Cannot recover lost semantic intent**, only restore structural consistency — data related to the interrupted operation may still effectively be lost, just no longer silently, dangerously inconsistent.
- **Must run before the file system can be safely used again** after an unclean shutdown, creating real, sometimes very long, downtime.

## Best Practices

- When explaining why a computer sometimes takes an unusually long time to boot after an improper shutdown, connect it directly to fsck's full-disk consistency scan.
- Recognize fsck's scaling problem (full-disk scan regardless of actual affected data) as the direct motivation for journaling (the next topic) — a fundamentally different strategy that avoids this scaling issue entirely.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A crash during a file system operation always causes permanent, unrecoverable data loss." | Structural inconsistencies (like an orphaned inode) can generally be detected and at least structurally repaired by fsck; the interrupted operation's specific intended result may not be recoverable, but the file system as a whole is not necessarily permanently broken. |
| "fsck fixes the exact problem that was created by the specific interrupted operation, restoring things to exactly what they would have been without the crash." | fsck can only detect and repair structural inconsistencies (bitmaps and directories disagreeing with each other, orphaned inodes, and similar) — it doesn't know what the interrupted operation was semantically trying to accomplish, so it repairs consistency, not necessarily intent. |

## Interview Questions

1. **Q: What is the crash consistency problem?**
   A: A file system operation (like creating a file) requires multiple separate disk writes; if a crash occurs between some of them completing and others not, the file system can be left in an inconsistent state where different structures (bitmaps, inodes, directories) disagree with each other.

2. **Q: What is an orphaned inode, and how does it arise?**
   A: An inode marked as "in use" in the inode bitmap, with valid metadata, but referenced by no directory entry anywhere — it arises when a crash occurs after an inode is allocated and initialized but before the corresponding directory entry is added.

3. **Q: What are fsck's two major limitations?**
   A: It must scan the entire disk to find and repair inconsistencies, which scales poorly as disk sizes grow — and it can only detect and repair structural inconsistencies, not necessarily recover the actual semantic intent of whatever operation was interrupted.

## Summary

- A single logical file system operation actually requires multiple separate disk writes, any of which a crash can interrupt between, potentially leaving different on-disk structures inconsistent with each other.
- A concrete example: a crash between initializing an inode and adding its directory entry produces an orphaned inode — marked in-use, but unreachable from any path.
- fsck detects and repairs such inconsistencies by scanning the entire disk after an unclean shutdown, but this scales poorly with disk size and cannot recover an interrupted operation's actual intent.
- The next topic covers journaling, the modern approach that prevents these inconsistencies from occurring in the first place, rather than detecting and repairing them after the fact.
