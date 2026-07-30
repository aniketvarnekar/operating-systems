# Module 20 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **On-Disk Structures: Inodes and Bitmaps** — the inode as per-file metadata and block pointers, bitmaps as compact free-space tracking
- [x] **Directories On Disk** — (name, inode number) directory entries, making hard links and deletion cost fully concrete
- [x] **Walking Through File Operations** — a complete, step-by-step trace of open()/read() and file creation

## The Big Picture

This module implemented Module 19's abstractions concretely, following the same "reveal the index structure underneath the clean abstraction" pattern Module 08 used for address spaces.

```
   Module 19: files & directories — the CLEAN ABSTRACTION
                          │
                          ▼
   Topic 1: inodes (per-file metadata + block pointers) +
            bitmaps (compact free-space tracking)
                          │
                          ▼
   Topic 2: directory entries = (name, inode number) —
            makes hard links and deletion cost fully concrete
                          │
                          ▼
   Topic 3: the FULL TRACE — every disk structure touched by
            open()/read() (lookups only) vs. file creation
            (genuine allocation across multiple structures)
                          │
                          ▼
   Module 21: FFS & LFS — designing these SAME structures
              around disk locality (Module 17, Topic 3)
```

## Practical Connections

- **Why "du" (disk usage) and "ls -i" (showing inode numbers) are standard UNIX diagnostic tools** — they directly expose the exact structures this module described: inode numbers and block allocation.
- **Why a file system can run out of the ability to create new files ("no space left on device," specifically for inodes) even when there's plenty of free data-block capacity remaining** — this is Topic 1's separate inode bitmap running out, independent of the data bitmap.
- **Why copying many small files is often slower, proportionally, than copying one large file of the same total size** — Topic 3's finding that file creation requires genuine allocation work (inode bitmap, directory entry) per file, not just per byte.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Inode vs. directory entry | An inode holds a file's actual metadata and data-block pointers. A directory entry is just a (name, inode number) pair — the name lives in the directory, not in the inode itself. |
| Inode bitmap vs. data bitmap | The inode bitmap tracks which inode numbers are free/in-use; the data bitmap separately tracks which data blocks are free/in-use — a file system can run out of one without running out of the other. |
| Reading vs. creating a file, in cost | Reading only requires looking up already-existing structures (inodes, directory entries). Creating requires allocating new resources across multiple structures (inode bitmap, inode table, directory data, eventually data bitmap) — genuinely more work. |

## What's Next

This module described a straightforward, if naive, on-disk layout — but didn't yet address Module 17, Topic 3's central finding: sequential disk access is dramatically faster than random access. **Module 21 — Fast File Systems and LFS** covers two real-world file system designs built specifically around that physical reality: FFS, which places related data physically close together, and log-structured file systems, which convert scattered writes into purely sequential ones.
