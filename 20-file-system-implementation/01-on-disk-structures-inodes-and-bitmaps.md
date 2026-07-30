# On-Disk Structures: Inodes and Bitmaps

## Learning Objectives

By the end of this section you should be able to:
- Explain what an inode is and what metadata it typically stores
- Explain how an inode's block pointers locate a file's actual data
- Explain how a bitmap tracks free space, for both inodes and data blocks

## Prerequisites

- Module 19, Topic 1 (The File Abstraction)
- Module 08, Topic 2 (Page Tables and Page Table Entries) — a useful structural parallel

## Motivation

Module 19 described files and directories from a program's perspective — clean, simple abstractions. This topic goes underneath: what actual data structures, stored on disk, does a file system use to make those abstractions real? Just as Module 08 revealed the page table underneath the address space abstraction, this topic reveals the inode underneath the file abstraction.

## Problem Statement

A file (Module 19, Topic 1) needs the OS to remember several things: how large it is, who owns it, what permissions apply to it, and — critically — exactly which physical disk blocks hold its actual data (which, exactly like a process's pages under paging, Module 08, Topic 1, need not be contiguous on disk at all). Where, on disk, is all of this per-file information itself stored?

## Concept

### The Inode

> An **inode** ("index node") is a fixed-size, per-file on-disk data structure holding a file's metadata: its size, owner, permissions, timestamps, and — most importantly — **pointers to the actual disk blocks holding its data**.

Every file (and every directory, Topic 2 — recall Module 19, Topic 2's "a directory is just a specially-interpreted file") has exactly one inode. Each inode is identified by a unique **inode number**, which is how the rest of the file system refers to a specific file internally (directory entries, Topic 2, map a human-readable name to an inode number, not to a filename stored anywhere else).

### Block Pointers: Locating a File's Data

Because a file's data can be scattered across non-contiguous disk blocks (exactly analogous to how a process's pages are scattered across non-contiguous physical frames, Module 08, Topic 1), the inode needs to record *which* specific blocks hold this file's data, and in what order.

A common approach: the inode directly stores a small number of **direct pointers**, each pointing to one specific data block — sufficient for small files. For larger files that need more blocks than the direct pointers alone can address, the inode also stores an **indirect pointer**, pointing not to a data block itself, but to an entire block *full of more pointers*, each of which points to an actual data block — and, for very large files, a **doubly indirect pointer** (a pointer to a block of pointers, each of which points to another block of pointers, each of which finally points to a data block), extending this same idea one level further.

```
   Inode for a file:
   ┌────────────────────────────────────┐
   │  size, owner, permissions, timestamps│
   ├────────────────────────────────────┤
   │  Direct pointer 1  → Data Block 12    │
   │  Direct pointer 2  → Data Block 47    │
   │  Direct pointer 3  → Data Block 3      │
   │  ... (a small, fixed number)           │
   │  Indirect pointer   → [ Block of MORE   │
   │                         pointers ] →     │
   │                         Data Block 900,   │
   │                         Data Block 12,     │
   │                         Data Block 501, ...│
   │  Doubly indirect ptr → ... (one more level)│
   └────────────────────────────────────┘
```

This design — small direct pointers for common, small files, with indirect pointers only needed for larger ones — mirrors the same "handle the common case cheaply, the rare case correctly" philosophy seen throughout this course (e.g., Module 09's multi-level page tables handling sparse address spaces).

### Bitmaps: Tracking Free Space

> A **bitmap** is a compact data structure using a single bit per tracked item (an inode, or a data block) to record whether that specific item is currently free or in use — one bitmap for inodes (tracking which inode numbers are available for a new file), and a separate bitmap for data blocks (tracking which physical blocks are available to hold data).

```
   Data block bitmap (1 bit per block):

   Block:   0  1  2  3  4  5  6  7  8  9
   Bitmap:  1  0  1  1  0  0  1  0  1  0
            (1 = in use, 0 = free)

   Creating a new file needing 2 blocks: scan for free (0) bits,
   e.g., blocks 1, 4 — mark them 1 (in use) once allocated
```

This is directly analogous to Module 08, Topic 1's observation that fixed-size chunks (pages/frames) enable simple, uniform free-space tracking (a bitmap) compared to the more complex variable-sized free-list management needed for segmentation (Module 07, Topic 3) — the same "fixed-size units → simple bitmap tracking" principle applies here to disk blocks.

## Internal Working (Preview)

```
   A simplified on-disk layout:

   ┌──────────┬───────────┬───────────┬───────────────────────┐
   │ Superblock │ Inode Bitmap │ Data Bitmap │ Inode Table │ Data Blocks │
   └──────────┴───────────┴───────────┴───────────────────────┘

   Superblock: overall file system metadata (total size, layout info)
   Inode Bitmap: which inode numbers are free/in-use
   Data Bitmap: which data blocks are free/in-use
   Inode Table: the actual fixed-size inode structures themselves
   Data Blocks: where actual file (and directory, Topic 2) contents live
```

## Real-World Analogy

Think of an inode like a library's catalog card for one specific book — it doesn't contain the book's actual text, but records the book's metadata (author, publication date, page count) and, critically, exactly which specific shelf locations hold each of its physical volumes (if the book is split across several volumes scattered on different shelves, exactly like a large file's data scattered across many disk blocks). A bitmap is like a simple checklist posted at the front desk, one checkbox per shelf slot in the entire library, marked simply "occupied" or "empty" — letting staff quickly find available shelf space for a new book's volumes without needing to walk the entire library checking each shelf individually.

## Why This Design Is Necessary

Just as a process needs a page table (Module 08, Topic 2) to record where its scattered pages actually live in physical memory, a file needs an inode to record where its scattered blocks actually live on physical disk — the file abstraction's transparency (Module 19, Topic 1) is only possible because this per-file bookkeeping exists somewhere, translating "the file's logical contents" into "these specific physical disk blocks." Bitmaps exist because fixed-size disk blocks (chosen specifically to avoid Module 07's fragmentation problems, echoing Module 08's paging rationale) can be tracked with extremely simple, compact, one-bit-per-block bookkeeping.

## Advantages of This Design

- **Uniform, fixed-size inode structures** make it simple to compute exactly where a specific inode number's data is stored on disk (a fixed offset calculation, similar in spirit to a page table's indexed lookup, Module 08, Topic 2).
- **Bitmaps are compact and simple** to scan for free space, directly analogous to why fixed-size pages enabled simpler free-space tracking than segmentation (Module 07, Topic 3 vs. Module 08, Topic 1).
- **Indirect pointers scale to large files** without requiring every inode to reserve space for an enormous number of direct pointers "just in case" — small files stay cheap, exactly mirroring Module 09, Topic 3's multi-level page table rationale.

## Disadvantages / Costs

- **Accessing a large file's later blocks requires an extra disk read** to fetch the indirect (or doubly indirect) pointer block itself, before the actual data block can be located — a real, if usually small, additional cost for large files, similar in spirit to Module 09, Topic 3's extra sequential reads for multi-level page table misses.
- **A fixed number of direct pointers imposes an implicit small-file "fast path" size**, beyond which the extra indirection overhead kicks in.

## Best Practices

- When explaining why very large files can have slightly slower random-access performance for their later portions, connect it directly to indirect/doubly-indirect pointer indirection requiring extra disk reads to locate the actual data block.
- Recognize the recurring "index structure translates a logical reference into a physical location" pattern across this course — page tables for memory (Module 08), inodes for files — as the same underlying idea applied to two different resources.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "An inode contains the file's actual data." | An inode contains metadata and pointers to the blocks that hold the actual data — the data itself lives in separate data blocks, referenced by the inode's direct/indirect pointers, not stored within the inode itself. |
| "A file system needs to scan every disk block individually to find free space." | A bitmap provides a compact, one-bit-per-block summary of free/in-use status, letting the file system find free space by scanning a small bitmap rather than inspecting every actual block's contents. |

## Interview Questions

1. **Q: What is an inode, and what does it store?**
   A: A fixed-size, per-file on-disk structure storing metadata (size, owner, permissions, timestamps) and pointers to the actual disk blocks holding the file's data — every file and directory has exactly one inode, identified by a unique inode number.

2. **Q: How do indirect pointers let an inode support files larger than its direct pointers alone could address?**
   A: An indirect pointer points not to a data block directly, but to a block full of additional pointers, each pointing to an actual data block — extending the addressable range without requiring every inode to reserve space for a huge number of direct pointers.

3. **Q: What is a bitmap used for in a file system, and why is it an efficient choice?**
   A: A bitmap tracks which inodes or data blocks are currently free versus in use, one bit per item. It's efficient because fixed-size blocks (chosen to avoid fragmentation) allow this extremely compact, simple representation, rather than the more complex variable-sized free-list tracking needed for non-fixed-size allocation schemes.

## Summary

- An inode is a per-file, fixed-size on-disk structure storing metadata and pointers (direct, indirect, doubly indirect) to the actual data blocks holding a file's contents.
- Bitmaps compactly track which inodes and data blocks are currently free or in use, exploiting fixed block sizes for simple, efficient free-space management.
- This mirrors Module 08's page table structure closely: both are index structures translating a logical reference (a virtual page; a file's logical position) into a physical location (a frame; a disk block).
- The next topic covers how Module 19, Topic 2's directory abstraction is actually implemented using these same inode and data-block structures.
