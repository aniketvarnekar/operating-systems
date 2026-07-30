# Module 20 — File System Implementation

## Module Goal

By the end of this module, you will understand **how a file system actually organizes physical disk blocks to implement the file and directory abstractions from Module 19** — inodes, bitmaps, and on-disk directory structures — and be able to walk through, step by step, what happens on disk when a file is created, read, or written.

## Topics Covered in This Module

1. **[On-Disk Structures: Inodes and Bitmaps](01-on-disk-structures-inodes-and-bitmaps.md)** — The inode as a file's on-disk metadata record, and bitmaps for tracking free space.
2. **[Directories On Disk](02-directories-on-disk.md)** — How Module 19's name-to-reference mappings are actually stored as inode-number lists.
3. **[Walking Through File Operations](03-walking-through-file-operations.md)** — A step-by-step trace of open(), read(), and file creation, tying every on-disk structure together.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 19 in full — this module implements the abstractions that module defined.
- Module 08, Topic 2 (Page Tables and Page Table Entries) — a useful structural parallel, since an inode plays a similar "per-object metadata and block-pointer record" role.

## How to Study This Module

Read in order. Topic 1 introduces the two core on-disk data structures — inodes (per-file metadata) and bitmaps (free-space tracking) — that everything else in this module is built from. Topic 2 shows how Module 19, Topic 2's directory abstraction is actually realized using these same structures — a directory's "contents" turn out to just be a list of (name, inode number) pairs. Topic 3 is the payoff: a complete, concrete trace of exactly which on-disk structures are read and written for real operations, tying Topics 1–2 together into one coherent picture.
