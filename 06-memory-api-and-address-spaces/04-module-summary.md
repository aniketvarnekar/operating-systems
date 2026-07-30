# Module 06 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Address Space Abstraction** — regions of an address space (Code, Static Data, Heap, Stack), and the three goals of memory virtualization (transparency, efficiency, protection)
- [x] **The Memory API** — stack vs. heap allocation, malloc/free/calloc/realloc, and how the heap actually grows via the OS
- [x] **Common Memory Bugs** — memory leaks, dangling pointers, double frees, buffer overflows, and uninitialized reads

## The Big Picture

This module began Virtualization's memory half by defining the abstraction (the address space) and the everyday API programs use within it (malloc/free). Just as Module 02 defined "process" before Modules 03–05 explained how the CPU is actually virtualized underneath it, this module defined "address space" before Modules 07–11 explain how memory is actually virtualized underneath it — translation, performance optimization, and handling memory pressure.

```
        MEMORY VIRTUALIZATION'S GOALS (Topic 1)
        ────────────────────────────────────────
        Transparency │ Efficiency │ Protection
              │             │            │
              ▼             ▼            ▼
        Every mechanism in Modules 07–11 is judged
        against these three goals:

        Module 07: Address Translation (base & bounds, segmentation)
        Module 08: Paging Fundamentals
        Module 09: Paging Performance (TLBs, multi-level tables)
        Module 10: Beyond Physical Memory (swapping)
        Module 11: Page Replacement Policies
```

## Practical Connections

- **Why a long-running server process's memory usage graph slowly creeps upward over days or weeks until it's restarted** — a classic, real-world symptom of a memory leak (Topic 3) that's easy to miss in a short test run but becomes obvious in production.
- **Why "use-after-free" and "buffer overflow" are among the most common vulnerability categories reported in security advisories for C/C++ software** — these are exactly the dangling-pointer and buffer-overflow bugs from Topic 3, which is precisely why memory-safe languages (Rust, and garbage-collected languages like Java, Python, Go) exist as deliberate design responses to this exact problem category.
- **Why a program's memory footprint doesn't shrink immediately after you delete a large data structure in some languages, but does in others** — this depends entirely on the underlying memory-management model: manual (Topic 2's malloc/free, immediate but error-prone) vs. automatic garbage collection (delayed, but eliminates several of Topic 3's bug classes by construction).

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Stack allocation vs. heap allocation | Stack: automatic, compiler-managed, tied to function-call lifetime. Heap: explicit, programmer-managed via malloc/free, persists until explicitly released. |
| malloc() vs. calloc() | malloc() makes no guarantee about the returned memory's contents; calloc() explicitly guarantees it's zeroed out. |
| Memory leak vs. dangling pointer | A leak is memory that's still valid but unreachable and never freed. A dangling pointer is a pointer to memory that HAS been freed, and using it accesses memory the allocator may have already reused. |
| Virtual address (Topic 1) vs. physical address | A process's own code only ever sees virtual addresses, starting at zero; the OS and hardware transparently translate these to real physical RAM locations, covered starting in Module 07. |

## What's Next

This module described the address space abstraction and the everyday API for using it — but deliberately left the actual mechanism unexplained: *how* does the OS transparently translate a process's private virtual addresses into real physical memory locations, satisfying the transparency, efficiency, and protection goals from Topic 1? **Module 07 — Address Translation** answers this, starting with the earliest, simplest schemes (base & bounds, segmentation) before Module 08 introduces the dominant modern technique: paging.
