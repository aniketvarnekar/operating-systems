# The Memory API

## Learning Objectives

By the end of this section you should be able to:
- Explain the difference between stack (automatic) and heap (dynamic) memory allocation
- Describe what malloc(), free(), calloc(), and realloc() each do
- Explain, at a high level, how the heap actually grows to accommodate more memory over a process's lifetime

## Prerequisites

- Topic 1 (The Address Space Abstraction)

## Motivation

Topic 1 described the heap as one of the regions in a process's address space, but didn't say how a program actually gets memory from it. This topic covers the concrete, everyday API — the one every C programmer calls directly, and the one that underlies the memory-management runtime of most higher-level languages — for requesting and releasing memory dynamically.

## Problem Statement

Some data a program needs has a size that isn't known until the program is actually running — for example, an array whose length depends on user input, or a data structure that needs to grow arbitrarily large based on how much data it's asked to hold. Local variables declared normally in a function are automatically placed on the stack (Module 02, Topic 1) and automatically reclaimed the instant that function returns — but stack space is managed automatically by the compiler, tied strictly to function-call lifetime, and isn't suited to data that needs to persist independently of any one function call, or whose size isn't known at compile time at all. Something else is needed: memory the program can explicitly request, of a size decided at runtime, that persists until the program explicitly says it's done with it.

## Concept

### Stack Allocation vs. Heap Allocation

| | Stack allocation | Heap allocation |
|---|---|---|
| **How it's requested** | Implicitly, by declaring a local variable | Explicitly, via a function call like `malloc()` |
| **Size known** | At compile time | Can be decided at runtime |
| **Lifetime** | Automatically tied to the enclosing function call | Persists until explicitly freed by the program |
| **Managed by** | The compiler (automatic) | The programmer (manual, in C) |

Heap allocation exists precisely to serve the cases stack allocation cannot: data whose size isn't known until runtime, or data that needs to outlive the specific function call that created it.

### The Core Memory API

> **`malloc(size)`** requests a block of heap memory of at least `size` bytes, and returns a pointer to the beginning of that block (or a special "null" value if the request could not be satisfied — e.g., the system is out of available memory).

> **`free(pointer)`** returns a previously-`malloc`'d block of memory back to the heap, marking it as available for a future allocation request to reuse. It takes only a pointer — the underlying allocator (a library that manages the heap on the program's behalf) is responsible for internally tracking how large that specific block actually was.

Two closely related functions round out the core API:

- **`calloc(count, size)`** allocates memory for `count` elements of `size` bytes each (equivalent to `count * size` total bytes), and additionally guarantees the memory is zeroed out before being returned — unlike plain `malloc()`, which makes no such guarantee and may return memory containing leftover, unpredictable data from whatever previously occupied it.
- **`realloc(pointer, new_size)`** resizes a previously allocated block to a new size, preserving as much of its existing contents as still fits, and returns a pointer to the (possibly relocated) block — since growing a block in place isn't always possible if adjacent memory is already in use, `realloc()` may need to allocate an entirely new, larger block elsewhere and copy the old data into it.

### How the Heap Actually Grows

The heap doesn't start out unlimited — it begins as a fairly small region, and the memory-allocation library (which implements `malloc`/`free` on the program's behalf) must ask the OS for more actual memory when it runs low on space to satisfy a request. This is done via a system call — traditionally `brk`/`sbrk` on UNIX-like systems, which asks the OS to move the boundary of the heap region further out — or, in modern practice, often via `mmap`, which can request an entirely separate region of memory from the OS as needed. Either way, the crucial point is: `malloc()` and `free()` themselves are ordinary library functions, not system calls — they manage memory the process already has, and only reach out to the OS (via an actual system call, Module 01, Topic 6) when they genuinely need to acquire more from it, or occasionally to release a large chunk back.

## Internal Working (Preview)

```
   Program calls malloc(64)
          │
          ▼
   malloc library checks its own internal bookkeeping:
   "do I already have 64+ free bytes available in the
    heap region the OS previously gave me?"
          │
     ┌────┴────┐
     ▼          ▼
    YES         NO
     │          │
     │          ▼
     │    ask the OS (via a system call, e.g. sbrk/mmap)
     │    for more actual memory, extending the heap
     │          │
     └────┬─────┘
          ▼
   return a pointer to a 64-byte block within
   the (possibly now-extended) heap region
```

## Real-World Analogy

Think of the heap like a self-storage facility, and `malloc()`/`free()` like the facility's front-desk staff. When you (the program) ask for a storage unit of a specific size (`malloc(size)`), the staff first checks whether they already have a suitably-sized empty unit available on-site; only if they don't do they go acquire an entirely new section of the building from the property owner (the OS, via a system call) to carve new units out of. When you're done with a unit, you hand back your key (`free(pointer)`) — the staff marks that specific unit as available again for a future customer, without needing you to remind them how big it was; they already have that on file internally.

## Why This Design Is Necessary

Manual, explicit heap allocation exists because the compiler alone cannot know, at compile time, exactly how much memory a program will need for data whose size genuinely depends on runtime conditions (user input, file contents, network data). Separating this from the automatic, compiler-managed stack lets each mechanism be optimized for what it's actually good at: the stack for fast, automatic, function-call-scoped storage; the heap for flexible, programmer-controlled, arbitrarily-sized and arbitrarily-lived storage.

## Advantages of This Design

- **Flexibility** — memory can be requested in exactly the size needed, decided at runtime, and can persist for as long as the program requires, independent of any single function call's lifetime.
- **Efficient reuse** — `free()`'d memory can be reused by later `malloc()` calls without repeatedly going back to the OS for more, since the allocator library manages its own already-acquired heap region internally.

## Disadvantages / Costs

- **Manual management burden** — in C, the programmer is entirely responsible for calling `free()` at the right time, exactly once, for every `malloc()`'d block — get this wrong, and you get exactly the bugs covered in the next topic (memory leaks, dangling pointers, double frees).
- **Real overhead versus stack allocation** — heap allocation/deallocation calls involve real bookkeeping work inside the allocator library (and occasionally a genuine system call to the OS), making them measurably more expensive than the compiler's near-free automatic stack management.

## Best Practices

- Always pair every `malloc()`/`calloc()` with exactly one corresponding `free()` — track ownership of allocated memory clearly (which part of the code is responsible for eventually freeing it) to avoid both leaks (never freed) and double-frees (freed more than once, covered next topic).
- Prefer `calloc()` over `malloc()` when zeroed-out memory is actually required, rather than manually zeroing it yourself afterward — it communicates intent clearly and can sometimes be implemented more efficiently by the allocator.
- Never assume `realloc()` returns the *same* pointer it was given — always reassign the result to a variable, since the block may have been relocated.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "malloc() and free() are themselves system calls." | They are ordinary library functions that manage memory the process already has; they only invoke an actual system call (e.g., sbrk/mmap) when they need to request additional memory from the OS, not on every single call. |
| "free(pointer) needs to be told how much memory to release, just like malloc() needed a size." | The allocator library internally tracks the size of every block it handed out; free() only needs the pointer, not the size, to correctly release the corresponding block. |
| "calloc(count, size) is just malloc(count * size) with extra steps." | calloc() additionally guarantees the returned memory is zeroed out; plain malloc() makes no such guarantee and may return memory still containing arbitrary leftover data. |

## Interview Questions

1. **Q: What's the difference between stack and heap memory allocation?**
   A: Stack allocation happens implicitly via local variable declarations, is managed automatically by the compiler, and is strictly tied to the enclosing function call's lifetime. Heap allocation happens explicitly via calls like malloc(), can be sized at runtime, and persists until the program explicitly frees it.

2. **Q: What does calloc() guarantee that malloc() does not?**
   A: calloc() guarantees the returned memory is zeroed out before use; malloc() makes no such guarantee and may return memory containing unpredictable leftover data from its previous use.

3. **Q: Are malloc() and free() themselves system calls?**
   A: No — they're ordinary library functions that manage a heap region the process has already been given. They only invoke an actual system call (like sbrk or mmap) when the allocator needs to request additional memory from the OS, not on every allocation or deallocation.

## Summary

- Stack allocation is automatic and compiler-managed, tied to function-call lifetime; heap allocation is explicit, programmer-managed, and can persist independently of any single function call.
- malloc(), free(), calloc(), and realloc() are the core memory API; calloc() additionally zeroes memory, and realloc() resizes a block, possibly relocating it.
- malloc()/free() are library functions, not system calls — they only reach out to the OS when the allocator's own existing heap region needs to grow.
- The next topic covers what goes wrong when this manual management is done incorrectly: memory leaks, dangling pointers, buffer overflows, and other classic bugs.
