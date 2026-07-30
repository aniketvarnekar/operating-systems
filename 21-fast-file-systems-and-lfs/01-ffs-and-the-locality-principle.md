# FFS and the Locality Principle

## Learning Objectives

By the end of this section you should be able to:
- Explain what problem the original naive file system layout had, motivating FFS
- Explain what a cylinder/block group is and how FFS uses it
- Explain the general locality principle FFS applies, and why it directly follows from Module 17, Topic 3

## Prerequisites

- Module 17, Topic 3 (Hard Disk Drive Mechanics)
- Module 20 in full

## Motivation

Module 20 described inodes, bitmaps, and directory entries as if their physical placement on disk didn't matter. It does — enormously, given Module 17, Topic 3's finding that seek time dominates disk access cost. This topic covers the Fast File System (FFS), one of the most influential real-world file system designs, built specifically to exploit that finding.

## Problem Statement

Imagine a naive file system layout where all inodes are grouped together in one region of the disk, and all data blocks are grouped together in a completely different, physically distant region (not unlike Module 20, Topic 1's simplified diagram). Reading a file requires reading its inode (in the inode region) and then its data (in the far-away data region) — for every single file access, the disk arm must seek a significant physical distance between these two regions. Could a smarter physical layout reduce this constant, expensive back-and-forth seeking?

## Concept

### The Locality Principle

> **FFS (the Fast File System)** is built around a single core idea: **place data that is likely to be accessed together physically close together on disk**, specifically to minimize the seek time (Module 17, Topic 3) between related accesses.

### Cylinder Groups (Block Groups)

> FFS divides the disk into several **cylinder groups** (also called block groups), each containing its own local pool of inodes, its own local bitmaps, and its own local data blocks — rather than one single, disk-wide pool of inodes far away from one single, disk-wide pool of data blocks.

```
   Naive layout (Module 20's simplified picture):

   [ ALL Inodes ][ ALL Data Blocks ]
    (far apart — every file access seeks across this whole gap)


   FFS layout: several smaller, self-contained CYLINDER GROUPS

   [ Group 1: inodes | bitmaps | data ][ Group 2: inodes | bitmaps | data ] ...

   (a file's inode and its OWN data blocks are placed within the
    SAME cylinder group whenever possible — MUCH shorter seeks)
```

### Applying Locality Within a Cylinder Group

FFS applies this same locality principle at a finer grain too:

- **A file's inode and its data blocks** are placed within the same cylinder group whenever possible, so reading a file's inode and then its data (Module 20, Topic 3's trace) requires only a short seek, not a long cross-disk one.
- **Files within the same directory** are placed within the same cylinder group as each other and as their containing directory, since files in the same directory are often accessed together in practice (e.g., listing a directory and then opening several of its files in sequence).

### The Trade-off: Large Files Must Still Span Groups

A single cylinder group has a limited amount of space — a very large file cannot fit entirely within one group. FFS handles this by spreading a very large file's data across multiple groups once it exceeds a threshold, accepting some seek cost for large files in exchange for keeping the (much more common) small-to-medium file case fast — directly mirroring the "optimize for the common case" philosophy seen throughout this course.

## Internal Working (Preview)

```
   Cylinder Group structure:

   ┌───────────────────────────────────────────┐
   │  Cylinder Group 1                              │
   │  ┌─────────┬───────────┬──────────────┐   │
   │  │ Inode Bmp │ Data Bmp    │ Inode Table    │   │
   │  ├─────────┴───────────┴──────────────┤   │
   │  │  Data Blocks (for files whose inodes    │   │
   │  │  live in THIS group)                     │   │
   │  └───────────────────────────────────────┘   │
   └───────────────────────────────────────────┘

   File's inode → placed HERE
   File's data  → placed HERE TOO (same group — SHORT seek)

   Files in the same directory → clustered in the SAME group
   as each other and their directory
```

## Real-World Analogy

Think of the naive layout like a library that keeps every single catalog card in one building across town from where all the actual books are shelved — every single lookup requires a long trip between the two buildings. FFS is like reorganizing the library into several smaller, self-contained branch libraries, each with its own catalog cards AND the actual books those cards refer to, kept in the same building — so looking up a card and then walking to grab its book is a short walk within one branch, not a long trip across town. Books that tend to be checked out together (like all the books in one specific course's reading list) are further kept on nearby shelves within the same branch, minimizing the walking distance for related lookups.

## Why This Design Is Necessary

Module 17, Topic 3 established that seek time is often the dominant cost in hard disk access — a physical, mechanical reality no amount of clever on-disk data structure design (Module 20) alone can avoid, if related data is scattered arbitrarily far apart. FFS directly attacks this by making physical placement a first-class design concern: rather than treating "where exactly on disk does this go" as an afterthought, it deliberately clusters data likely to be accessed together, converting what would otherwise be many long, expensive seeks into many short, cheap ones.

## Advantages of FFS's Locality-Based Design

- **Dramatically reduced seek time for the common case** — reading a file's inode and data, or browsing files within one directory, typically stays within one cylinder group.
- **Directly and deliberately exploits Module 17, Topic 3's physical reality**, rather than ignoring it the way a naive layout does.

## Disadvantages / Costs

- **Large files must still span multiple cylinder groups**, incurring some seek cost — an accepted trade-off for keeping the common (small-to-medium file) case fast.
- **As a disk fills up and cylinder groups become full**, FFS may be forced to place new data in a less-local group than ideal, gradually eroding some of the locality benefit over the disk's lifetime (a phenomenon sometimes mitigated by keeping some free space reserved specifically to preserve placement flexibility).

## Best Practices

- When explaining why file system design cares so much about physical layout (not just logical structure), lead directly with Module 17, Topic 3's seek-time finding — FFS's entire design is a direct engineering response to that one physical fact.
- Recognize the "cluster related things physically close together, accept some cost for the less-common large/spread-out case" pattern as a recurring theme, echoing the direct/indirect pointer split from Module 20, Topic 1.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "FFS's cylinder groups are just a different way of organizing the same total disk space, with no real performance benefit." | The entire point is physical placement: keeping a file's inode and data (and files within the same directory) physically close together directly minimizes seek time, a real, measurable performance benefit given Module 17, Topic 3's mechanical realities. |
| "FFS eliminates seek time entirely." | It minimizes seek time for the common case (related data clustered together), but large files spanning multiple groups, or accesses to genuinely unrelated files in different groups, still incur real seek costs — FFS reduces, but does not eliminate, this mechanical cost. |

## Interview Questions

1. **Q: What is the core design principle behind FFS?**
   A: Placing data likely to be accessed together — a file's inode and its own data blocks, and files within the same directory — physically close together on disk, specifically to minimize seek time given hard disk drives' mechanical realities (Module 17, Topic 3).

2. **Q: What is a cylinder group, and how does it differ from a naive "all inodes here, all data there" layout?**
   A: A cylinder group is a self-contained region of the disk with its own local inodes, bitmaps, and data blocks. Unlike a naive layout that separates all inodes from all data blocks (requiring long seeks for every access), a cylinder group keeps a file's inode and its own data physically close together.

3. **Q: How does FFS handle files too large to fit within a single cylinder group?**
   A: It spreads such a file's data across multiple cylinder groups once it exceeds a threshold, accepting some additional seek cost for large files in exchange for keeping the far more common small-to-medium file case fast.

## Summary

- FFS is built around locality: placing data likely to be accessed together physically close on disk, directly minimizing seek time given Module 17, Topic 3's mechanical realities.
- Cylinder groups keep a file's inode and its own data blocks (and files within the same directory) physically close together, rather than separating all inodes from all data disk-wide.
- Large files must still span multiple groups, an accepted trade-off for keeping the common case fast.
- The next topic covers a more radical response to the same physical realities: log-structured file systems, which convert nearly all writes into one purely sequential stream.
