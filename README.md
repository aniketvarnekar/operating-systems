# The Complete Operating Systems Course — From First Principles to Mastery

> A self-contained Operating Systems learning resource. No prior OS knowledge assumed.
> Written to replace textbooks, university slides, and scattered blog posts for learning OS concepts deeply.

## Who this is for

- Complete beginners to Operating Systems (but you should already be comfortable writing programs in *some* language, and know what a variable, a function call, and a program's execution flow are)
- College students studying OS for coursework or exams
- Backend/systems developers who use processes, threads, and files daily but never learned the machinery underneath them
- Engineers preparing for technical interviews (deadlock, virtual memory, and scheduling questions show up constantly)
- Anyone who wants to understand what actually happens between "I ran a program" and "a program ran" — not just use an OS, but *understand* it

## Philosophy

Most OS resources teach you **definitions**. This course teaches you **reasoning**.

For every concept, we don't just tell you what it's called — we explain:

| Question | Why it matters |
|---|---|
| **Why does this exist?** | Every OS abstraction (processes, virtual memory, files) was invented to solve a real, painful problem with raw hardware. If you don't know the problem, the abstraction feels arbitrary and you'll forget it. |
| **What problem does it solve?** | Concepts stick when tied to a concrete pain point — "what goes wrong without this?" |
| **How does it work internally, mechanically?** | An OS is built from a small number of repeating tricks (virtualization, concurrency control, indirection, persistence via logging). Once you see the trick once, you recognize it everywhere. |
| **When should I reach for which policy? What are the trade-offs?** | Almost every OS design decision (which scheduler, which page-replacement policy, which file system) is a trade-off between competing goals. Nothing is free. |
| **What actually goes wrong in practice?** | Race conditions, deadlocks, thrashing, and crash inconsistency are not academic — they are real bugs in real systems. |

We follow this learning loop for every topic:

```
   Problem
      │
      ▼
    Need
      │
      ▼
  Concept
      │
      ▼
Mechanism (How)
      │
      ▼
Policy (What/When)
      │
      ▼
 Trade-offs
      │
      ▼
Common Failures
      │
      ▼
Best Practices
```

This mirrors how the field itself is usually taught: **separate the mechanism from the policy**. The mechanism is the low-level "how" (e.g., how a context switch physically happens); the policy is the higher-level "what should we do" (e.g., which process should run next). Keeping these two questions distinct is the single biggest idea that makes the rest of OS design click.

## How this course is generated

This course is built **one module at a time**. At the start of a module, every topic it will cover is listed up front. At the end of a module, every topic is checked off, and a summary ties it back to earlier and later modules. The next module only begins when you ask for it (type **"Continue"**).

This keeps each module focused, complete, and reviewable before moving on — rather than dumping an entire course at once.

## Course Structure (24 Modules)

The course follows the natural shape of the field: every operating system topic is really an instance of one of three big ideas — **making a scarce physical resource look infinite (Virtualization)**, **coordinating things that happen at the same time (Concurrency)**, and **making data outlive the process that created it, even across a crash (Persistence)**.

| # | Module | Status | What you'll learn |
|---|---|---|---|
| 01 | [Introduction](01-introduction/) | ✅ Complete | What an OS is, the three big ideas (virtualization, concurrency, persistence), a brief history, system calls previewed |
| | **— Virtualization: the CPU —** | | |
| 02 | [Processes](02-processes/) | ✅ Complete | The process abstraction, process states, the process control block, the fork/exec/wait/exit API |
| 03 | [Direct Execution](03-direct-execution/) | ✅ Complete | Limited direct execution, user vs. kernel mode, traps and trap tables, context switching |
| 04 | [CPU Scheduling Basics](04-cpu-scheduling-basics/) | ✅ Complete | Scheduling metrics, FIFO, SJF, STCF, Round Robin, mixing CPU and I/O bursts |
| 05 | [CPU Scheduling Advanced](05-cpu-scheduling-advanced/) | ✅ Complete | Multi-Level Feedback Queue, lottery/proportional-share scheduling, multi-CPU scheduling |
| | **— Virtualization: Memory —** | | |
| 06 | [Memory API & Address Spaces](06-memory-api-and-address-spaces/) | ✅ Complete | malloc/free, common memory bugs, the address space abstraction and its goals |
| 07 | [Address Translation](07-address-translation/) | ✅ Complete | Base & bounds (dynamic relocation), segmentation, free-space management |
| 08 | [Paging Fundamentals](08-paging-fundamentals/) | ✅ Complete | Page tables, address translation via paging, page table entries and protection bits |
| 09 | [Paging Performance](09-paging-performance/) | ✅ Complete | TLBs, multi-level page tables, inverted page tables |
| 10 | [Beyond Physical Memory](10-beyond-physical-memory/) | ✅ Complete | Swapping mechanism, swap space, the page-fault handling path |
| 11 | [Page Replacement Policies](11-page-replacement-policies/) | ✅ Complete | Optimal, FIFO, LRU and its practical approximations (Clock), thrashing |
| | **— Concurrency —** | | |
| 12 | [Concurrency: Threads](12-concurrency-threads/) | ✅ Complete | Why threads exist, the thread abstraction, the thread API, common creation patterns |
| 13 | [Locks](13-locks/) | ✅ Complete | Critical sections, building locks in hardware (test-and-set, compare-and-swap), spinlocks, ticket locks, futex-based locks |
| 14 | [Condition Variables & Semaphores](14-condition-variables-and-semaphores/) | ✅ Complete | Producer/consumer, condition variables, semaphores as a unifying primitive, dining philosophers |
| 15 | [Concurrency Bugs](15-concurrency-bugs/) | ✅ Complete | Deadlock (conditions, prevention, avoidance, detection), atomicity and order-violation bugs |
| 16 | [Event-Based Concurrency](16-event-based-concurrency/) | ✅ Complete | select/poll/epoll, event loops, the blocking-call problem |
| | **— Persistence —** | | |
| 17 | [I/O Devices & Disks](17-io-devices-and-disks/) | ✅ Complete | The canonical device protocol, device drivers, hard disk drive mechanics, disk scheduling |
| 18 | [RAID](18-raid/) | ✅ Complete | RAID levels 0/1/4/5 and the capacity/reliability/performance trade-off triangle |
| 19 | [Files & Directories](19-files-and-directories/) | ✅ Complete | The file and directory abstractions, hard vs. soft links, mounting |
| 20 | [File System Implementation](20-file-system-implementation/) | ✅ Complete | A simple file system's on-disk structures (inodes, bitmaps, data blocks), operation walkthroughs |
| 21 | [Fast File Systems & LFS](21-fast-file-systems-and-lfs/) | ✅ Complete | Locality-aware file system design (FFS), log-structured file systems |
| 22 | [Crash Consistency](22-crash-consistency/) | ✅ Complete | The crash-consistency problem, fsck, write-ahead journaling |
| 23 | [Distributed Systems & Integrity](23-distributed-systems-and-integrity/) | ✅ Complete | Data integrity via checksums, an introduction to networked file systems (NFS, AFS) |
| | **— Wrap-up —** | | |
| 24 | [Interview Preparation](24-interview-preparation/) | ✅ Complete | Consolidated, ranked interview questions and mock scenarios across all modules |

## Companion References (grow with every module)

- **[interview-questions.md](interview-questions.md)** — consolidated interview Q&A across all modules

## How to read this course

1. Go in order. Modules build on each other — Module 08 (Paging) assumes Module 06 (Address Spaces); Module 15 (Concurrency Bugs) assumes Modules 12–14.
2. Trace every diagram with your finger/cursor — don't skim ASCII diagrams, they encode real state transitions.
3. Attempt exercises and "predict what happens" prompts *before* checking the explanation that follows.
4. Keep the mechanism/policy distinction in mind at all times — it's the single idea that makes OS design stop feeling like a pile of unrelated trivia.
5. Revisit the "Why" sections even if you already know "How" — most working developers can *use* processes, threads, and files, but can't explain *why* the OS is built this way, and that gap shows up in interviews.
