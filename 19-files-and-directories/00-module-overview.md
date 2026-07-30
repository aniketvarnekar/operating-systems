# Module 19 — Files and Directories

## Module Goal

By the end of this module, you will understand **the file and directory abstractions themselves** — what programs actually interact with, sitting above the physical storage mechanisms Modules 17–18 covered — including hard and soft links, and how multiple physical storage devices are unified into a single logical directory hierarchy via mounting.

## Topics Covered in This Module

1. **[The File Abstraction](01-the-file-abstraction.md)** — What a file actually is from the OS's perspective, and the API (open, read, write, close) programs use to interact with one.
2. **[The Directory Abstraction](02-the-directory-abstraction.md)** — How files are named and organized hierarchically, and what a directory actually contains internally.
3. **[Hard Links, Soft Links, and Mounting](03-hard-links-soft-links-and-mounting.md)** — Multiple names for the same underlying file, symbolic references, and unifying multiple physical volumes into one directory tree.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 06, Topic 1 (The Address Space Abstraction) — a useful point of comparison, since files play a similar "clean abstraction over dangerous raw hardware" role for persistent storage.
- Module 17–18 — the physical storage this module's abstractions sit on top of.

## How to Study This Module

Read in order. Topic 1 defines the file itself and the fundamental API every program uses to interact with one — deliberately mirroring Module 06, Topic 1's abstraction-goals framing, but for persistent storage instead of memory. Topic 2 covers how files get organized and found at all (directories), including a detail that trips up many beginners: a directory is itself just a specially-interpreted file. Topic 3 covers the more advanced naming and organization features (links, mounting) built on top of Topics 1–2's foundation.
