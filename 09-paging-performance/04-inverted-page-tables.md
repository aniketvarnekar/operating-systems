# Inverted Page Tables

## Learning Objectives

By the end of this section you should be able to:
- Explain what makes an inverted page table "inverted" relative to a normal page table
- Explain why an inverted page table's size scales with physical memory rather than virtual address space
- Explain the search-cost trade-off inverted page tables introduce, and how hashing addresses it

## Prerequisites

- Module 08, Topic 2 (Page Tables and Page Table Entries)
- Topic 3 (Multi-Level Page Tables) — helpful context, not required

## Motivation

Topic 3 solved the space problem by restructuring the page table into a tree, still fundamentally organized "one entry (or sub-table) per virtual page." This topic covers a fundamentally different approach to the exact same space problem: instead of organizing the table around the (potentially enormous) virtual address space, organize it around the (typically much smaller) physical memory instead.

## Problem Statement

Even a well-designed multi-level page table (Topic 3) still allocates second-level tables proportional to how much of the virtual address space is actively in use — and on a system running many processes, each with a reasonably large *active* footprint, this can still add up to a meaningful amount of per-process bookkeeping overhead. Physical memory, meanwhile, is a comparatively small, fixed resource: a machine might have a vast, sparse virtual address space per process, but only a modest, fixed number of physical frames in total, shared across every process combined. Could the OS get away with a table sized according to that smaller, shared, fixed quantity instead?

## Concept

### The Core Idea: One Entry Per Physical Frame, Not Per Virtual Page

> An **inverted page table** maintains exactly **one entry per physical frame** in the entire system (not one entry per virtual page, and not one table per process) — each entry records which process, and which of that process's virtual pages, currently occupies that specific physical frame.

This is precisely why it's called "inverted": a normal page table (flat or multi-level) is indexed by virtual page number and stores a physical frame number. An inverted page table flips this relationship — it's indexed by physical frame number and stores (effectively) a virtual page number, plus which process it belongs to.

### Why This Saves Space

Because there is only **one** inverted page table for the **entire system** — shared across every process — its total size is proportional to the amount of physical memory the machine actually has (a fixed, comparatively modest number of frames), rather than proportional to the sum of every process's virtual address space size (Topic 3's multi-level approach, still ultimately bounded by how much of each process's *virtual* space is in active use). A machine with a modest amount of physical RAM has an inverted page table bounded by that same modest size, regardless of how many processes are running or how large their individual virtual address spaces theoretically are.

### The Cost: Searching Becomes Harder

A normal page table (flat or multi-level) is naturally indexed *by* the very thing you're translating (the VPN) — looking up a specific virtual page's translation is a direct, fast lookup. An inverted page table is indexed the *other* way (by physical frame), which means a translation request (which naturally starts with a VPN, for a specific process) requires **searching** the inverted table to find the one entry (if any) matching that process and VPN — a linear scan of every single physical frame's entry would be far too slow to be practical.

> Real implementations address this by maintaining an auxiliary **hash table**, keyed by (process ID, VPN), pointing directly to the matching entry in the inverted page table — turning what would otherwise be a slow linear search into a fast, near-direct hash lookup, at the cost of maintaining this additional hash structure alongside the inverted table itself.

## Internal Working (Preview)

```
 Normal (multi-level) page table: one structure PER PROCESS,
 sized according to that process's VIRTUAL address space usage

   Process A's tables        Process B's tables        ...
   (sized to A's usage)      (sized to B's usage)


 Inverted page table: ONE structure for the WHOLE SYSTEM,
 sized according to PHYSICAL memory (frame count)

   Frame #    Process    VPN
   ───────   ────────   ─────
     0          A          12
     1          B           3
     2          A           0
     3        (free)        —
     ...
   N-1          B          47

   Translating (Process A, VPN 12) requires finding WHICH FRAME
   entry matches — a hash table keyed by (process, VPN) makes
   this a fast lookup instead of a slow linear scan.
```

## Real-World Analogy

Think of a normal (even multi-level) page table like each hotel guest keeping their own personal notebook listing which of their belongings are in which room. An inverted page table is like the hotel instead keeping just **one single master log, one line per physical room in the building**, recording which guest (and which of their labeled items) currently occupies each room — this master log is naturally sized to the hotel's actual, fixed room count, not to the sum of every guest's own itemized packing list. The catch: if a guest asks "which room is my blue suitcase in," the hotel can't just look it up directly in a log organized by room number — it needs a separate, fast cross-reference index (the hash table) mapping "guest + item name" straight to the correct room-log entry, or staff would have to check every single room's log entry one by one to find the match.

## Why This Approach Is Useful

Inverted page tables are a genuine, historically significant alternative design point specifically for systems where minimizing total page-table memory overhead (across potentially many processes with large virtual address spaces) matters more than the simplest possible translation logic — trading a fast, direct virtual-address-indexed lookup for a much smaller total structure, recovering search speed via an auxiliary hash table rather than through the table's own indexing scheme.

## Advantages of Inverted Page Tables

- **Table size scales with physical memory, not virtual address space** — a single, system-wide structure whose size is bounded by the machine's actual, fixed frame count, regardless of how many processes run or how large their individual address spaces are.
- **One structure for the whole system** rather than a separate structure per process, which can further reduce total overhead on a system running many processes simultaneously.

## Disadvantages / Costs

- **Naive lookup requires a search, not a direct index** — since the table is organized by physical frame, not by the (process, VPN) pair a translation request actually starts with, a direct correspondence doesn't exist without an auxiliary structure.
- **Requires maintaining an additional hash table** to make lookups practically fast — extra bookkeeping and complexity compared to a normal page table's naturally direct indexing.

## Best Practices

- When comparing page-table designs, always frame the choice as a trade-off between **what the table's size scales with** (virtual address space for normal/multi-level tables, vs. physical memory for inverted tables) and **how directly a translation request maps to a lookup** (direct indexing vs. requiring an auxiliary hash structure) — these two axes are what distinguish the approaches, not one being simply "better" in isolation.
- Recognize inverted page tables as a genuine, real-world design used on some architectures specifically because physical memory is a much more tightly bounded resource than the sum of many processes' virtual address spaces — not a purely theoretical alternative.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "An inverted page table is just a multi-level page table with an extra step." | It's a fundamentally different organizing principle: multi-level tables are still indexed by virtual page number (just hierarchically); an inverted table is indexed by physical frame number instead, requiring an auxiliary hash structure to support fast virtual-address-based lookups at all. |
| "Inverted page tables always outperform multi-level page tables since there's only one table for the whole system." — | "One table" being smaller in total doesn't automatically mean faster lookups; without the auxiliary hash table, a naive inverted-table lookup would require an impractical search across every physical frame's entry — the space savings come with a genuine, separate lookup-cost trade-off to manage. |

## Interview Questions

1. **Q: What makes an inverted page table "inverted" compared to a normal page table?**
   A: A normal page table is indexed by virtual page number and stores a physical frame number. An inverted page table flips this: it's indexed by physical frame number, and each entry records which process and virtual page currently occupies that frame.

2. **Q: Why does an inverted page table's size scale with physical memory rather than virtual address space?**
   A: Because there is exactly one entry per physical frame in the entire system, and only one such table shared across all processes — its size is bounded by the machine's fixed physical frame count, regardless of how many processes are running or how large their individual virtual address spaces are.

3. **Q: What cost does an inverted page table introduce, and how is it typically addressed?**
   A: Translating a (process, virtual page) pair requires finding the matching entry in a table organized by physical frame instead — a naive linear search would be impractically slow. Real implementations maintain an auxiliary hash table, keyed by (process ID, VPN), to make this lookup fast.

## Summary

- An inverted page table has one entry per physical frame (not per virtual page), recording which process and virtual page currently occupies each frame — the opposite indexing direction from a normal page table.
- Its size scales with physical memory (a fixed, comparatively modest, system-wide quantity) rather than with the sum of every process's virtual address space usage.
- The cost is that translation requests (which start with a process and VPN) require an auxiliary hash table to avoid an impractically slow linear search across the inverted table.
- This closes out the module's coverage of paging performance — the module summary ties the TLB, ASIDs, multi-level tables, and inverted tables together as the complete answer to Module 08's two cliffhangers, before Module 10 addresses what happens when physical memory itself runs out entirely.
