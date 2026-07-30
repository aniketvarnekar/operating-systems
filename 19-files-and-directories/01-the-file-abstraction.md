# The File Abstraction

## Learning Objectives

By the end of this section you should be able to:
- Define a file precisely, as the OS's abstraction over persistent storage
- Describe the core file API: open, read, write, close, and what a file descriptor is
- Explain why file operations must go through system calls

## Prerequisites

- Module 01, Topic 5 (Persistence)
- Module 01, Topic 6 (System Calls and the OS Interface)
- Module 06, Topic 1 (The Address Space Abstraction) — a useful structural parallel

## Motivation

Modules 17–18 covered the physical reality underneath persistent storage: devices, disk mechanics, and RAID. But no program directly manipulates raw disk sectors — it interacts with **files**. This topic defines that abstraction precisely, mirroring how Module 06, Topic 1 defined the address space abstraction for memory.

## Problem Statement

A disk (Module 17, Topic 3) is physically just a large sequence of numbered sectors. Directly manipulating raw sector numbers would require every program to know the exact physical layout of whatever storage device it happens to be running on — exactly the kind of dangerous, unmanageable complexity Module 06, Topic 1 identified for raw physical memory. What abstraction does the OS provide instead, letting programs work with named, organized, persistent data without any awareness of physical sector numbers?

## Concept

### Definition

> A **file** is the OS's abstraction for a named, persistent sequence of bytes, stored on non-volatile storage (Module 01, Topic 5), that a program can create, read, write, and delete without any awareness of the physical disk sectors actually involved — the OS (specifically, the file system, Module 20) handles that translation entirely.

This is a direct, deliberate parallel to Module 06, Topic 1's address space: exactly as a process gets a private, simple, zero-based view of memory regardless of physical RAM location, a program gets a simple, named, byte-sequence view of persistent data regardless of physical disk sector location.

### The Core File API

Programs interact with files through a small set of system calls (Module 01, Topic 6):

> **open(path, flags)**: locates (or creates) the file at the given path and returns a **file descriptor** — a small integer the OS uses internally to refer to this specific, now-open file for all subsequent operations.

> **read(fd, buffer, size)**: reads up to `size` bytes from the file referenced by `fd`, starting from its current position, into `buffer`.

> **write(fd, buffer, size)**: writes `size` bytes from `buffer` into the file referenced by `fd`, at its current position.

> **close(fd)**: tells the OS this program is done with the file, releasing the file descriptor and any associated OS resources.

### The File Descriptor and the Current Position

Every open file has an associated **current position** (sometimes called a file offset) — a marker tracking where the *next* read or write will occur, automatically advanced after each operation. This is precisely the "open file descriptor" state that Module 02, Topic 5 mentioned survives an `exec()` call, and that Module 02, Topic 7 showed being manipulated by a shell to implement I/O redirection *before* calling `exec()`.

### Why File Operations Must Go Through System Calls

Exactly as Module 06, Topic 1 established for memory, direct access to a disk's physical sectors is a restricted operation (Module 03, Topic 2) — a user process cannot simply read or write raw disk hardware itself. Every file operation must trap into the kernel (Module 01, Topic 6), which validates the request (does this program actually have permission to access this specific file?) and, working through the file system (Module 20) and the device driver (Module 17, Topic 2), performs the actual physical disk access on the program's behalf.

## Internal Working (Preview)

```
   Program's view:                        What actually happens:

   fd = open("notes.txt", ...)      ──►    trap into kernel (Module 01,
                                             Topic 6); file system (Module 20)
                                             locates the file's actual disk
                                             blocks; returns a file descriptor

   read(fd, buffer, 100)            ──►    trap into kernel; file system
                                             translates "the next 100 bytes
                                             of this file" into actual disk
                                             sector reads (via the device
                                             driver, Module 17, Topic 2);
                                             advances fd's current position

   close(fd)                        ──►    trap into kernel; releases the
                                             file descriptor and associated
                                             OS resources
```

## Real-World Analogy

Think of a file like a labeled folder in a large archive building, and the file descriptor like a small numbered claim ticket a librarian hands you the moment you request that folder (open()). You never need to know which physical shelf, room, or storage unit the folder actually lives on — you simply present your claim ticket to read the next page (read()) or add a new page (write()), and the librarian (the OS, via the file system) handles the actual physical retrieval and storage entirely behind the scenes. When you're finished, you hand back the claim ticket (close()), and the librarian is free to reuse that ticket number for someone else's future request.

## Why This Design Is Necessary

Without the file abstraction, every program would need direct, specific knowledge of physical disk sector numbers and layout — an unmanageable, device-specific burden exactly analogous to what Module 06, Topic 1 identified for raw physical memory addresses. The file abstraction, backed by the same system-call/trap mechanism (Module 01, Topic 6) that protects memory and process management, lets programs work with simple, named, byte-sequence data while the OS handles the messy physical reality (Modules 17–18) underneath, transparently.

## Advantages of the File Abstraction

- **Complete transparency from physical storage details** — a program never needs to know or care about disk sectors, device types, or RAID configuration (Module 18) underneath a file it's using.
- **A small, simple, uniform API** (open/read/write/close) that works identically regardless of the underlying storage hardware or file system implementation.

## Disadvantages / Costs

- **Every file operation costs a real system call** (Module 01, Topic 6), with the associated trap overhead — programs performing many small, individual file operations pay this cost repeatedly, which is why buffering (batching many small operations into fewer, larger ones) is a common performance optimization.

## Best Practices

- Always pair every open() with a corresponding close(), exactly as Module 06, Topic 2 emphasized pairing every malloc() with a free() — an unclosed file descriptor is a real, if usually smaller-scale, resource leak.
- Recognize the file descriptor as playing a role directly analogous to a process's PID (Module 02, Topic 3) or a memory allocation's pointer (Module 06, Topic 2) — a small handle the OS uses to track a specific, live resource on your program's behalf.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A file's contents are stored at one single, predictable location the program can compute directly." | A file's actual physical location (which disk sectors hold its data) is determined and managed entirely by the file system (Module 20); the program only ever sees a simple byte-sequence abstraction via its file descriptor, never the physical layout. |
| "read() and write() operate independently of any file position tracking." | Every open file maintains a current position, automatically advanced after each read/write, which is exactly why sequential reads/writes on the same file descriptor naturally continue from where the last operation left off. |

## Interview Questions

1. **Q: What is a file, in the OS sense?**
   A: The OS's abstraction for a named, persistent sequence of bytes stored on non-volatile storage, which a program can create, read, write, and delete without any awareness of the actual physical disk sectors involved.

2. **Q: What is a file descriptor, and what does open() return?**
   A: A small integer the OS uses to refer to a specific open file for all subsequent operations. open() returns this file descriptor after locating (or creating) the requested file.

3. **Q: Why must file operations go through system calls rather than direct disk access?**
   A: Direct physical disk access is a restricted operation; a user process cannot perform it itself. Every file operation must trap into the kernel, which validates the request and performs the actual disk access (via the file system and device driver) on the program's behalf.

## Summary

- A file is the OS's abstraction for a named, persistent byte sequence, hiding physical disk sector details entirely — a direct parallel to how an address space hides physical memory location.
- The core file API (open, read, write, close) operates on a file descriptor, with every open file tracking a current position that automatically advances.
- Every file operation requires a system call, since direct physical disk access is restricted to the kernel.
- The next topic covers how files are actually named and organized — the directory abstraction — including the detail that a directory is itself just a specially-interpreted file.
