# Module 19 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The File Abstraction** — files as a named, persistent byte-sequence abstraction, and the open/read/write/close API
- [x] **The Directory Abstraction** — directories as specially-interpreted files containing name-to-reference mappings, and path resolution
- [x] **Hard Links, Soft Links, and Mounting** — multiple names for one file, path-based indirect references, and unifying separate devices into one directory tree

## The Big Picture

This module built the file-system abstraction layer directly on top of Modules 17–18's physical foundation, following the exact same transparency pattern established for memory back in Module 06.

```
   Module 06, Topic 1: address space — hides physical RAM location
                          │  (the SAME transparency goal, now applied
                          │   to persistent storage)
                          ▼
   Topic 1: the FILE — hides physical disk sector location
                          │
                          ▼
   Topic 2: the DIRECTORY — a specially-interpreted FILE, mapping
            names to references; path resolution walks this structure
                          │
                          ▼
   Topic 3: hard links (multiple names, one file), soft links
            (path-based indirection), mounting (multiple physical
            DEVICES → one seamless tree)
                          │
                          ▼
   Module 20: how a real file system implements ALL of this
              on actual physical disk blocks
```

## Practical Connections

- **Why deleting a file you've hard-linked elsewhere doesn't actually free up disk space until every link is removed** — this is Topic 3's reference-counting mechanism, directly observable behavior on any real UNIX-like system.
- **Why a broken desktop shortcut (or a broken symlink) can point at "nothing" after you move or rename the original file** — this is Topic 3's soft-link dangling-reference risk, experienced firsthand.
- **Why plugging in an external drive makes its files "just appear" under a folder like `/Volumes` or `/mnt`, with no special commands needed to browse them** — this is Topic 3's mounting mechanism, made concrete and visible.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| A file vs. a directory | A directory is not a fundamentally different kind of object — it's a file whose contents are specially interpreted as a list of name-to-reference mappings, rather than arbitrary program data. |
| Hard link vs. soft link | A hard link directly shares the same underlying file and reference count. A soft link is an independent file storing a path, resolved separately, and can dangle if that path becomes invalid. |
| Deleting a file's name vs. deleting its data | Deleting a directory entry (a name) only removes that specific reference; the underlying data persists until its reference count reaches zero, i.e., until every hard link to it is gone. |

## What's Next

This module described the file and directory abstractions from the perspective of a program using them. **Module 20 — File System Implementation** goes one level deeper: how a real file system actually organizes physical disk blocks to implement these abstractions — inodes, bitmaps, on-disk directory structures — and walks through exactly what happens, block by block, when a file is created, read, or written.
