# Directories On Disk

## Learning Objectives

By the end of this section you should be able to:
- Explain precisely how a directory's (name, reference) pairs from Module 19 are represented on disk
- Trace how a directory's own inode fits into this picture
- Explain why deleting a directory entry is a cheap operation, distinct from deleting the underlying file's data

## Prerequisites

- Topic 1 (On-Disk Structures: Inodes and Bitmaps)
- Module 19, Topic 2 (The Directory Abstraction)

## Motivation

Module 19, Topic 2 established that a directory is a specially-interpreted file containing (name, reference) pairs. Topic 1 of this module introduced the inode as the on-disk structure representing any file. This topic connects the two directly: what, concretely, is the "reference" half of a directory's (name, reference) pairs?

## Problem Statement

Module 19, Topic 2 was deliberately abstract about what a directory's "reference" actually is. Now that Topic 1 has introduced the inode and inode number as the concrete way the file system identifies any specific file, what exactly does a directory store to point at "the file named notes.txt"?

## Concept

### Directory Entries: (Name, Inode Number) Pairs

> A directory's on-disk contents — recall, a directory is itself just a file (Module 19, Topic 2), with its own inode (Topic 1) like any other — are a list of **directory entries**, each pairing a human-readable **name** with the **inode number** of the file (or sub-directory) that name refers to.

```
   Directory "/home/user/"'s contents (as data blocks, pointed to by
   ITS OWN inode, exactly like any other file):

   ┌───────────────┬──────────────┐
   │  Name             │  Inode Number    │
   ├───────────────┼──────────────┤
   │  notes.txt        │  1042             │
   │  photos            │  2001             │
   │  ..                 │  17  (parent dir)│
   └───────────────┴──────────────┘
```

This makes Module 19, Topic 2's path resolution completely concrete: resolving `/home/user/notes.txt` means reading the root directory's data blocks to find the entry named "home" (getting *its* inode number), using that inode to locate and read the "home" directory's own data blocks to find the entry named "user" (getting *its* inode number), and so on — at every step, following a (name → inode number) lookup, then using that inode number (via Topic 1's inode table) to find where the *next* directory's actual contents live on disk.

### Hard Links, Revisited Concretely

Module 19, Topic 3 described a hard link as "an additional directory entry pointing at the same underlying file." Now this is fully concrete: a hard link is simply **a second (name, inode number) entry — in the same or a different directory — using the identical inode number** as an existing entry. Since both entries share the same underlying inode, and the inode itself (Topic 1) tracks a reference count, this is exactly why deleting one directory entry (one name) doesn't touch the underlying file's data at all unless it was the very last entry referencing that inode number.

### Why Deleting a Directory Entry Is Cheap and Distinct From Deleting Data

Removing a file "deletion," from the file system's perspective, is really two conceptually separate steps:

1. **Remove the (name, inode number) pair** from the containing directory's data blocks — a small, cheap update to just that one directory's contents.
2. **Decrement the target inode's reference count** (Topic 1) — and only if that count reaches zero, actually mark the inode and its data blocks as free again (updating the bitmaps from Topic 1), making that space available for a future file.

This two-step structure is precisely why "deleting" a file with existing hard links elsewhere is cheap and doesn't touch the actual data at all — only step 1 happens for that specific name, while step 2's "reclaim the space" branch is skipped entirely, since the reference count hasn't reached zero.

## Internal Working (Preview)

```
   Creating a hard link "/backup/report.txt" for existing
   "/docs/report.txt" (inode 500):

   /docs/  directory's data:      /backup/  directory's data:
   ┌──────────┬─────┐            ┌──────────┬─────┐
   │report.txt  │ 500  │            │report.txt  │ 500  │  ← SAME inode number
   └──────────┴─────┘            └──────────┴─────┘

   Inode 500's reference count: 2 (was 1)


   Deleting "/docs/report.txt":

   /docs/  directory's data:      /backup/  directory's data:
   ┌──────────┬─────┐            ┌──────────┬─────┐
   │ (entry REMOVED)   │            │report.txt  │ 500  │  ← still valid!
   └──────────┴─────┘            └──────────┴─────┘

   Inode 500's reference count: 1 (decremented, NOT zero)
   → inode 500 and its data blocks remain FULLY INTACT
```

## Real-World Analogy

Recall Module 19, Topic 3's warehouse name-tag analogy — this topic makes it fully concrete: the "box" is an inode, identified by its catalog number (inode number), and each name tag is literally a (name, catalog-number) entry written on some drawer's index card (a directory's data). Removing one name tag from one drawer's index card is a trivially small edit to that one card — it has no effect on the actual box in the warehouse, or on any *other* drawer's index card that happens to reference the same catalog number, until every single index-card reference to that catalog number has been removed.

## Why This Design Is Necessary

Separating "remove a name" from "reclaim the underlying storage" is exactly what makes hard links (Module 19, Topic 3) work correctly and cheaply: if deleting a directory entry always immediately freed the underlying inode and data blocks, a file with multiple hard links could have its data destroyed the instant *any one* of its names was removed — breaking the other, still-valid names that point to the identical data. The reference-count mechanism (Topic 1's inode metadata) is precisely what allows the file system to know, safely and cheaply, exactly when it's actually safe to reclaim an inode's space.

## Advantages of This Design

- **Cheap, localized deletion** — removing one directory entry only requires updating that one directory's own data blocks, not touching the target file's inode or data at all (beyond a reference-count decrement).
- **Directly enables correct, safe hard links** — multiple names can legitimately share one inode, with automatic, correct cleanup only once the last reference is gone.

## Disadvantages / Costs

- **Reference counting requires careful, consistent bookkeeping** — an inode's count must be incremented and decremented correctly on every link creation and removal; a bug here could either leak space (count never reaches zero when it should) or corrupt data (count reaches zero while a directory entry still references it) — a correctness-critical detail explored further in Module 22's crash-consistency discussion.

## Best Practices

- When explaining "why doesn't deleting this file actually free up disk space," always check whether other hard links to the same inode still exist — the reference count, not the single deleted name, determines whether the underlying data is actually reclaimed.
- Keep the two-step deletion model (remove directory entry; conditionally reclaim inode/data) explicit in your mental model — conflating them is a common source of confusion about hard-link behavior.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A directory entry stores the target file's full metadata (size, permissions) directly." | A directory entry stores only a name and an inode number — all of the file's actual metadata lives in the inode itself (Topic 1), looked up via that inode number, not duplicated into every directory entry that references it. |
| "Deleting a file always immediately reclaims its disk space." | Deletion only removes one directory entry and decrements the target inode's reference count; the actual data and inode are only reclaimed once that count reaches zero, i.e., once every hard link to the file has been removed. |

## Interview Questions

1. **Q: What does a directory entry actually store on disk?**
   A: A name paired with an inode number — the actual file's metadata and data-block pointers live in the inode itself, looked up via that inode number, not duplicated in the directory entry.

2. **Q: How does this on-disk structure make hard links concrete?**
   A: A hard link is simply a second (name, inode number) entry — in the same or a different directory — using the same inode number as an existing entry, since both names share the identical underlying inode and its reference count.

3. **Q: Why is deleting a directory entry a cheap, localized operation distinct from reclaiming disk space?**
   A: Deletion only removes the (name, inode number) pair from the containing directory's data and decrements the target inode's reference count; the underlying inode and data blocks are only actually marked free (updating Topic 1's bitmaps) once that count reaches zero.

## Summary

- A directory's on-disk contents are a list of (name, inode number) pairs — directory entries — making Module 19's abstract "reference" concrete.
- A hard link is simply two directory entries sharing the same inode number; deletion only removes one entry and decrements the shared inode's reference count.
- The underlying data is only actually reclaimed (bitmaps updated, Topic 1) once an inode's reference count reaches zero — cheap, localized deletion that correctly supports hard links.
- The next topic ties Topics 1–2 together into a complete, step-by-step trace of what actually happens on disk during real file operations: opening, reading, and creating a file.
