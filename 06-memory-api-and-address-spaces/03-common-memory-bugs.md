# Common Memory Bugs

## Learning Objectives

By the end of this section you should be able to:
- Identify and explain memory leaks, dangling pointers, buffer overflows, uninitialized reads, and double frees
- Explain the concrete, observable symptom each bug tends to produce
- Explain why these specific bugs are a direct consequence of manual memory management

## Prerequisites

- Topic 2 (The Memory API)

## Motivation

Manual memory management (Topic 2) hands enormous flexibility to the programmer — and enormous responsibility along with it. This topic catalogs the classic mistakes that flexibility makes possible. These bugs are worth knowing precisely, not just by name, because they're some of the most common real-world sources of crashes, security vulnerabilities, and "works on my machine but not in production" mysteries in systems software history.

## Problem Statement

Because `malloc()` and `free()` (Topic 2) place the entire burden of correctness on the programmer — deciding when memory is needed, when it's safe to release, and exactly how much of it is valid to use — there are several distinct, well-known ways this can go wrong. Each one has a different root cause and a different observable symptom, and conflating them (as beginners often do) makes debugging real code much harder than it needs to be.

## Concept

### Memory Leaks

> A **memory leak** occurs when a program allocates memory via `malloc()` (or equivalent) and never calls `free()` on it, even though the program no longer has any way to reach or use that memory again.

The memory isn't corrupted or accessed incorrectly — it's simply never returned to the heap for reuse. A single leak is often harmless in a short-lived program (the OS reclaims *all* of a process's memory the instant it exits, leak or not), but in a long-running program (a server, a background service), leaks accumulate over time, gradually consuming more and more memory until performance degrades or the process eventually crashes from resource exhaustion.

### Dangling Pointers

> A **dangling pointer** is a pointer that still holds the address of memory that has already been `free()`'d — using it afterward (reading it, writing through it, or even calling `free()` on it again) accesses memory the allocator now considers available for a completely different, future allocation.

This is dangerous specifically because the memory hasn't necessarily been erased or made obviously invalid — it might still contain the old data, and might work correctly for a while, right up until that same memory gets handed out again to a genuinely different allocation, at which point using the dangling pointer silently corrupts unrelated data.

### Double Frees

> A **double free** happens when `free()` is called more than once on the same pointer.

This corrupts the allocator's own internal bookkeeping structures (the data it uses to track which blocks are free and which are in use), which can cause unpredictable, hard-to-diagnose failures far removed — in both time and code location — from the actual double-free call itself.

### Buffer Overflows (Overruns)

> A **buffer overflow** (or buffer overrun) occurs when a program writes past the boundary of an allocated block of memory, into adjacent memory it was never given permission to use.

This is one of the most historically significant bug classes in all of computing security: writing past a buffer's boundary can silently corrupt unrelated data sitting adjacent to it in memory, or — in a maliciously crafted case — overwrite critical control data (like a saved return address on the stack), potentially letting an attacker redirect a program's execution to code of their own choosing.

### Uninitialized Reads

> An **uninitialized read** occurs when a program reads from memory that was allocated (via `malloc()`, not `calloc()`) but never explicitly written to first, retrieving whatever arbitrary, leftover data happened to already be sitting in that memory.

Because plain `malloc()` makes no guarantee about the contents of newly allocated memory (Topic 2), this leftover data could be anything — including, in a security-sensitive context, fragments of another program's previously-freed data, if the OS's memory-reuse mechanisms happen to expose it.

## Internal Working (Preview)

```
 Memory leak:                          Dangling pointer:

 ptr = malloc(100);                    ptr = malloc(100);
 // ... ptr goes out of scope,          free(ptr);
 //     never freed ...                 // ... later ...
                                        *ptr = 42;   ← writes into memory
 (block is now unreachable,               the allocator may have already
  permanently "lost" until the            handed out to someone else
  process exits)

 Double free:                          Buffer overflow:

 ptr = malloc(100);                    char buf[10];
 free(ptr);                            buf[15] = 'X';  ← writes 6 bytes past
 free(ptr);   ← corrupts the             the allocated 10-byte block,
   allocator's own internal              into adjacent memory
   free-list bookkeeping
```

## Real-World Analogy

Think of the self-storage facility analogy from Topic 2. A **memory leak** is renting a unit, losing the paperwork for which unit it was, and never notifying the front desk you're done with it — the unit sits permanently reserved and unusable by anyone else, even though you yourself have no way to get back into it either. A **dangling pointer** is continuing to use your old key on a unit *after* you've already returned it and the facility has re-rented it to a new customer — you might still be able to unlock it, but whatever you find (or leave) inside now belongs to, and corrupts, someone else's storage. A **double free** is returning the same key to the front desk twice, confusing their records about which units are actually available. A **buffer overflow** is stacking boxes so far past the boundary of your own rented unit that they spill into and crush whatever your neighbor has stored in the adjacent unit.

## Why These Specific Bugs Exist

Every one of these bugs is a direct, structural consequence of the design established in Topic 2: `malloc()`/`free()` require the programmer to manually track exactly which memory is currently valid to use, exactly once, for exactly as long as it's needed — with no automatic enforcement of any of these rules by the language or runtime itself (unlike languages with automatic garbage collection, which eliminate leaks-via-forgetting and dangling pointers by design, at the cost of the performance/control trade-offs discussed elsewhere). C's manual model trades safety for control and performance — these bugs are the concrete cost of that trade-off when the manual bookkeeping goes wrong.

## Advantages of Understanding These Bugs Precisely

- **Faster, more accurate debugging** — recognizing a program's specific symptom (a slow memory-usage climb over time vs. a sudden crash vs. security-relevant memory corruption) lets you immediately narrow down which of these distinct bug categories you're likely dealing with.
- **Better code review instincts** — knowing exactly what conditions produce each bug lets you spot risky patterns (a pointer used after being freed, a buffer write without a bounds check) directly in source code, before they ever manifest as a runtime failure.

## Disadvantages / Real Dangers

- **Buffer overflows in particular have historically been a major real-world security vulnerability class**, enabling attackers to hijack a program's execution flow by carefully overwriting control data adjacent to an overflowed buffer.
- **These bugs are often non-deterministic in their symptoms** — a dangling pointer or buffer overflow might work "fine" for a long time before ever causing a visible failure, purely depending on what the corrupted memory happens to be used for afterward, making them notoriously hard to reliably reproduce and diagnose.

## Best Practices

- Always pair every `malloc()` with a clear, traceable plan for exactly one corresponding `free()` call, and set a pointer to a recognizable "empty" value immediately after freeing it, so an accidental subsequent use is easier to catch rather than silently becoming a dangling-pointer bug.
- Always validate that a write stays within an allocated buffer's actual size before performing it — never assume a fixed-size buffer will never receive more data than expected, especially for anything derived from external input.
- Use memory-debugging tools (e.g., tools that detect leaks, overflows, and use-after-free at runtime) during development rather than relying purely on manual code review to catch these bugs.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A memory leak will eventually cause my program to crash immediately." | A leak simply means memory is never reclaimed for reuse; for a short-lived program, or a program with plenty of spare memory, a leak may never cause a visible problem at all — but it remains a real bug that becomes serious in long-running programs as leaked memory accumulates over time. |
| "A dangling pointer is safe to use as long as I don't reallocate anything in between." | It's using memory that the allocator has already marked as available — even without a new allocation reusing that exact memory yet, the access is undefined behavior; relying on "it happened to still work" is not a guarantee of correctness. |
| "Buffer overflows are just a performance or correctness nuisance, not a serious concern." | Beyond corrupting adjacent data, buffer overflows are one of the most historically exploited security vulnerability classes, capable of letting an attacker redirect a program's execution entirely, in the worst case. |

## Interview Questions

1. **Q: What is a memory leak, and why is it more dangerous in a long-running program than a short-lived one?**
   A: A memory leak is allocated memory that's never freed, even though the program can no longer reach it. A short-lived program's entire memory is reclaimed by the OS on exit regardless, so a leak there is often harmless; a long-running program accumulates leaked memory over time, eventually degrading performance or exhausting available memory.

2. **Q: What's the difference between a dangling pointer and a double free?**
   A: A dangling pointer is a pointer still referencing memory that has already been freed, and using it (reading, writing, or freeing again) accesses memory the allocator may have already reused elsewhere. A double free is specifically calling free() twice on the same pointer, which corrupts the allocator's own internal bookkeeping of which blocks are free.

3. **Q: Why are buffer overflows considered a serious security vulnerability, not just a correctness bug?**
   A: Writing past a buffer's boundary can overwrite adjacent, unrelated memory — including critical control data like a saved return address — potentially letting a maliciously crafted overflow redirect a program's execution to code of the attacker's choosing.

4. **Q: What's the difference between malloc() and calloc() regarding uninitialized reads?**
   A: malloc() makes no guarantee about the contents of newly allocated memory, so reading it before writing can expose arbitrary leftover data. calloc() explicitly guarantees the returned memory is zeroed out, eliminating this specific risk.

## Summary

- Memory leaks (never freeing reachable-but-unused memory), dangling pointers (using memory after it's freed), double frees (freeing the same memory twice), buffer overflows (writing past an allocation's boundary), and uninitialized reads (reading never-written memory) are the classic bugs manual memory management makes possible.
- Each has a distinct root cause and a distinct observable symptom, which is the key to diagnosing which one you're dealing with in practice.
- These bugs are a direct, structural consequence of Topic 2's manual-management model — the price of the control and performance that model provides.
- This closes out the module's introduction to memory virtualization at the application level — the module summary ties the address space abstraction (Topic 1) and this API (Topics 2–3) together before Module 07 begins the actual address-translation mechanisms.
