# Walking Through File Operations

## Learning Objectives

By the end of this section you should be able to:
- Trace, step by step, every on-disk structure touched when opening and reading an existing file
- Trace, step by step, every on-disk structure touched when creating a brand-new file
- Explain why file creation touches meaningfully more on-disk structures than a simple read

## Prerequisites

- Topic 1 (On-Disk Structures: Inodes and Bitmaps)
- Topic 2 (Directories On Disk)

## Motivation

Topics 1–2 introduced the individual pieces — inodes, bitmaps, directory entries. This topic is the payoff: tying every piece together into one concrete, complete trace of what actually happens, on disk, for the operations Module 19, Topic 1 introduced as a clean, simple API.

## Problem Statement

When a program calls `open("/home/user/notes.txt")` followed by `read()`, how many actual disk structures does the file system need to consult, and in what order? And when a program instead calls something that creates a brand-new file, what *additional* work does the file system need to do, beyond what a simple read requires?

## Concept

### Tracing open() and read() for an Existing File

Recall path resolution (Module 19, Topic 2), now made fully concrete with Topic 2's (name, inode number) directory entries:

1. **Read the root directory's inode** (its location is fixed/known by convention) to find its data blocks.
2. **Read the root directory's data blocks**, searching for the entry named "home" — find its inode number.
3. **Read the "home" directory's inode** (using the inode number just found) to locate *its* data blocks.
4. **Read the "home" directory's data blocks**, searching for "user" — find its inode number.
5. **Read the "user" directory's inode**, then **its data blocks**, searching for "notes.txt" — find *its* inode number.
6. **Read notes.txt's own inode** — this is what `open()` ultimately returns a file descriptor referencing.
7. When `read()` is subsequently called: **consult notes.txt's inode's block pointers** (Topic 1) to find which actual data block(s) hold the requested bytes, then **read those data blocks**.

Notice how many separate disk reads a single `open()` on a deeply nested path actually requires — one inode read plus one data-block read for *every* directory level in the path, before even reaching the target file's own inode. This is precisely why real file systems aggressively **cache** recently-accessed inodes and directory data in memory (an idea directly analogous to the TLB, Module 09, Topic 1, caching recently-used translations) — repeatedly re-walking the same path from scratch on every single access would be prohibitively slow otherwise.

### Tracing File Creation

Creating a brand-new file requires meaningfully more work than reading an existing one, because several on-disk structures must be **allocated and updated**, not just read:

1. **Resolve the containing directory's path** (steps 1–5 above, ending at the containing directory's own inode and data).
2. **Find a free inode number** — consult the inode bitmap (Topic 1), find a free (0) bit, and mark it in-use (1).
3. **Initialize the new inode** — write its metadata (size 0 initially, owner, permissions, timestamps) into the inode table at that inode number's location.
4. **Add a new directory entry** — write a new (name, inode number) pair into the containing directory's own data blocks (Topic 2), potentially requiring the directory itself to grow (needing a new data block allocated for it, via the data bitmap, if its existing blocks are already full).
5. Only once the program actually **writes data** to the new file does the file system need to allocate actual **data blocks** for that content — consulting the data bitmap (Topic 1), marking blocks as in-use, and updating the new inode's block pointers to reference them.

## Internal Working (Preview)

```
   open("/home/user/notes.txt") — READING an existing file:

   read root inode → read root's data (find "home" → inode #A)
   read inode #A   → read #A's data   (find "user" → inode #B)
   read inode #B   → read #B's data   (find "notes.txt" → inode #C)
   read inode #C   ← THIS is what the file descriptor refers to

   read(fd, ...) afterward:
   consult inode #C's block pointers → read the actual data block(s)


   CREATING a new file "/home/user/new.txt":

   [ resolve /home/user/ exactly as above, ending at its inode + data ]
        │
        ▼
   1. Inode Bitmap: find a free inode number (say, #D) → mark IN-USE
   2. Inode Table:  initialize inode #D (size=0, owner, perms, ...)
   3. /home/user/'s data blocks: ADD entry ("new.txt", #D)
        (may require allocating a NEW data block for the directory
         itself, via the Data Bitmap, if it's already full)
   4. (later, on first write) Data Bitmap: find free block(s) →
        mark IN-USE, update inode #D's block pointers to reference them
```

## Real-World Analogy

Reading an existing file is like following a chain of catalog cards through a library — root catalog → wing catalog → shelf catalog → the specific book's own card — each step requiring you to pull and read one more card before you know where to look next, until you finally reach the book itself. Creating a brand-new file is like registering an entirely new book with the library for the first time: you need to find and claim an unused catalog number (the inode bitmap), fill out a fresh catalog card for it (initializing the inode), add its name to the appropriate shelf's index (the directory entry) — possibly needing to add an entirely new index page if the shelf's existing index is already full — and only once you actually place physical pages on a shelf (writing data) does the library need to find and claim actual shelf space for those pages (the data bitmap).

## Why File Creation Is Meaningfully More Expensive Than Reading

Reading an existing file only ever needs to **look up** already-established structures — inodes and directory entries that already exist, requiring no allocation decisions at all. Creating a file requires **allocating** new resources (a free inode, a directory entry, and eventually data blocks) — decisions that must consult and update the bitmaps (Topic 1) and correctly modify the containing directory's own data (Topic 2), any one of which could, in principle, be interrupted by a crash partway through — a concern this module deliberately sets aside, but which Module 22 (Crash Consistency) addresses directly and rigorously.

## Advantages of Understanding This Full Trace

- **Explains real, observable performance differences** — why creating many small files can be noticeably slower than reading the same number of already-existing files, since creation involves genuine allocation work, not just lookups.
- **Sets up Module 22 precisely** — by walking through this multi-step sequence explicitly, you can now see exactly *where* a crash occurring mid-sequence (e.g., after allocating an inode but before adding its directory entry) could leave the file system in an inconsistent, confusing state — the exact problem Module 22 is built to solve.

## Disadvantages / Costs

- **Deep path resolution genuinely costs multiple disk reads** — a good reason real file systems cache recently-accessed inodes and directory data aggressively in memory, rather than re-walking the full path from the root on every single access.
- **File creation touches multiple distinct on-disk structures** (inode bitmap, inode table, containing directory's data, and eventually the data bitmap) — each an opportunity for inconsistency if interrupted, motivating Module 22's crash-consistency techniques.

## Best Practices

- When reasoning about file system performance, distinguish "pure lookup" costs (reading existing structures, mitigated heavily by caching) from "allocation" costs (creating/modifying structures, which cannot be avoided by caching alone).
- Keep this topic's full multi-step trace in mind heading into Module 22 — it's the concrete, step-by-step picture that "what if a crash happens between step 3 and step 4" questions are actually asking about.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Opening a file only requires one disk read — to fetch the file's own inode." | Opening a file by path requires resolving every directory level named in that path first, requiring one inode read and one data-block read per level, before finally reaching the target file's own inode. |
| "Creating a new file is essentially the same cost as reading an existing one, just with a different starting point." | File creation requires genuine allocation across multiple structures (a free inode from the bitmap, an initialized inode, a new directory entry, and eventually data blocks) — meaningfully more work than the pure lookups a read of an existing file requires. |

## Interview Questions

1. **Q: Walk through what happens, on disk, when a program calls open() on a nested file path.**
   A: The file system reads the root directory's inode and data to find the next path component's inode number, then repeats this (read inode, read its data, find the next component) for every directory level in the path, until finally reading the target file's own inode — which is what the returned file descriptor references.

2. **Q: What additional steps does creating a new file require, beyond what reading an existing file needs?**
   A: Finding and marking a free inode number in the inode bitmap, initializing that inode's metadata, adding a new (name, inode number) entry to the containing directory's data (possibly requiring a new data block for the directory itself), and eventually allocating data blocks (via the data bitmap) once actual content is written.

3. **Q: Why do real file systems cache recently-accessed inodes and directory data in memory?**
   A: Because path resolution requires reading an inode and directory data block for every level of a path, repeatedly re-walking the same path from the root on every single access would be prohibitively slow — caching avoids re-reading structures that were just recently accessed, analogous in spirit to the TLB's caching of recent address translations.

## Summary

- Opening a file requires resolving every directory level in its path — one inode read and one data-block read per level — before finally reaching the target file's own inode.
- Creating a new file requires genuine allocation: a free inode (via the inode bitmap), an initialized inode, a new directory entry (possibly growing the directory itself), and eventually data blocks (via the data bitmap).
- File creation is meaningfully more expensive than reading, since it involves allocation decisions across multiple on-disk structures, not just lookups.
- This closes out the module's implementation-level coverage of file systems — the module summary ties inodes, bitmaps, directory entries, and this operational trace together before Module 21 covers how real file systems (FFS, log-structured file systems) are specifically designed around Module 17, Topic 3's disk-locality realities.
