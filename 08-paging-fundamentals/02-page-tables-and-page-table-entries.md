# Page Tables and Page Table Entries

## Learning Objectives

By the end of this section you should be able to:
- Explain what a page table is and why every process needs its own
- Split a virtual address into a virtual page number and an offset, and explain what each part is used for
- List the typical fields inside a page table entry, beyond just the physical frame number

## Prerequisites

- Topic 1 (The Core Idea of Paging)

## Motivation

Topic 1 established that pages can map to frames arbitrarily, scattered anywhere in physical memory — but that flexibility only works if the OS reliably remembers exactly *which* frame each page currently maps to. This topic covers the data structure that remembers this, and precisely what it stores.

## Problem Statement

If Page 0 of a process maps to Frame 12, Page 1 to Frame 3, and Page 2 to Frame 47 (Topic 1's example), something must record these specific mappings — and record them separately for every process, since two different processes' Page 0 will almost certainly map to two entirely different physical frames. Without this record, the hardware would have no way to translate a virtual address into the correct physical one on any given memory access.

## Concept

### The Page Table

> A **page table** is a per-process data structure, maintained by the OS, that records the mapping from every virtual page in that process's address space to the physical frame it currently resides in.

Every process has its own separate page table — exactly as every process has its own PCB (Module 02, Topic 3) — reflecting the fact that each process's virtual pages map to entirely independent physical locations.

### Splitting a Virtual Address: Virtual Page Number + Offset

Just as a segmented address split into a segment number and an offset (Module 07, Topic 2), a paged virtual address splits into two parts:

```
 Virtual address:

  ┌─────────────────────┬──────────────┐
  │   Virtual Page Number  │    Offset      │
  │       (VPN)             │                │
  └─────────────────────┴──────────────┘
```

- The **Virtual Page Number (VPN)** identifies *which page* this address falls within — it's used to look up the correct entry in the page table.
- The **offset** identifies the specific byte *within* that page — since pages are fixed-size (Topic 1), the offset's bit-width is fixed too (e.g., a 4 KB page needs exactly 12 offset bits, since 2^12 = 4096).

The page table lookup uses the VPN to find the corresponding **Physical Frame Number (PFN)**; the offset is then simply copied unchanged, since it identifies the same relative position within whichever frame the page maps to:

```
 physical_address = (page_table[VPN].frame_number) ++ offset
                      (frame number replaces the VPN;
                       offset bits pass through unchanged)
```

### What a Page Table Entry (PTE) Actually Contains

A **page table entry (PTE)** is far more than just a physical frame number. Typical fields include:

- **Physical Frame Number (PFN)**: the actual translation target — which physical frame this virtual page currently maps to.
- **Valid bit**: indicates whether this page table entry represents a currently valid mapping at all — a process's address space typically has large unused regions (recall the heap/stack gap, Module 06, Topic 1), and there's no reason to waste a physical frame on a page that isn't actually in use; an access to an invalid page traps into the OS instead of translating to a nonsensical or dangerous physical address.
- **Protection bits**: specify what kind of access is permitted on this page — read, write, and/or execute — directly generalizing segmentation's per-segment protection bits (Module 07, Topic 2) down to a much finer, per-page granularity.
- **Present bit** (relevant starting in Module 10): indicates whether this page is currently resident in physical memory at all, or has been temporarily moved out to disk.
- **Reference and dirty bits** (relevant in Module 11): track whether a page has recently been accessed, and whether it's been modified since being loaded — information page-replacement policies use to make good decisions about which pages to evict when physical memory runs low.

## Internal Working (Preview)

```
   Virtual Address (VPN = 2, offset = 0x1A4)

        Page Table (this process's own)
        ┌─────┬──────┬───────┬────────────┐
        │ VPN  │ Valid │ Prot.  │ Frame #     │
        ├─────┼──────┼───────┼────────────┤
        │  0    │  1     │ R-X    │   12         │
        │  1    │  1     │ RW-    │   3           │
        │  2    │  1     │ RW-    │   47   ◄──────┤ lookup here
        │  3    │  0     │  --     │   —           │  (VPN 3 unused
        └─────┴──────┴───────┴────────────┘   at all — invalid)

   physical_address = 47 ++ 0x1A4   (frame number replaces VPN;
                                      offset passes through unchanged)
```

## Real-World Analogy

Think of a page table like a hotel's master room-assignment ledger for one specific, ongoing group booking (one process). Each entry in the ledger records, for a given internal group-member number (the VPN), which actual physical hotel room number they're assigned to (the frame number) — plus additional notes: whether that group-member number is even currently assigned to anyone at all (the valid bit — some numbers in the group's block might simply be unused), and what that specific person is allowed to do in their room (protection bits — some rooms might be marked staff-access-only, others guest-accessible). The room number *within* the building (the offset) stays meaningful regardless of which physical room you're assigned — "the window side of the room" means the same relative thing whether you're in room 12 or room 47.

## Why Page Table Entries Need More Than Just a Frame Number

A raw frame number alone would tell the hardware *where* a page's data is, but not *whether* that mapping is legitimately in use at all (the valid bit), nor *what* operations are permitted on it (protection bits) — both of which are essential for satisfying the protection goal from Module 06, Topic 1, at a much finer granularity than segmentation's per-segment-only protection (Module 07, Topic 2). Additional bits like present, reference, and dirty exist specifically to support later mechanisms (swapping, Module 10; page replacement, Module 11) that need to reason about a page's history and current residency, not just its current translation target.

## Advantages of This Design

- **Fine-grained protection** — read/write/execute permissions can be set independently for every single small page, not just per entire segment.
- **Supports sparse address spaces efficiently** — the valid bit means unused regions of a process's address space (like the heap/stack gap, Module 06, Topic 1) simply have no valid page table entries at all, consuming no physical frames for pages that were never actually used.

## Disadvantages / Costs (Previewed)

- **Every process needs its own complete page table**, covering every possible virtual page in its address space — and as Topic 3 of this module will show, a naive, single flat table covering an entire, large address space can itself consume a surprising amount of memory, even for pages that are marked invalid.

## Best Practices

- When reasoning about "why did this memory access fail," check the valid bit and protection bits conceptually first — many real-world crashes (segmentation faults, access violations) are precisely the hardware detecting an invalid or disallowed page table entry during translation.
- Keep the VPN/offset split distinct from the segment-number/offset split (Module 07, Topic 2) in your mental model: pages are uniformly small and fixed-size, so the split point is always the same fixed number of bits, unlike segmentation's variable-sized regions.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A page table entry only needs to store a physical frame number." | Real PTEs also store a valid bit (is this mapping in use at all), protection bits (what access is allowed), and additional bits (present, reference, dirty) used by later mechanisms like swapping and page replacement. |
| "Every possible virtual page in a process's address space has a meaningful, resident physical frame." | Most address spaces have large unused regions; their corresponding page table entries are simply marked invalid, and no physical frame is ever allocated for them. |

## Interview Questions

1. **Q: What is a page table, and why does every process need its own?**
   A: A per-process data structure recording the mapping from every virtual page to its current physical frame. Every process needs its own because each process's virtual pages map to entirely independent physical locations, exactly as each process has its own address space (Module 06, Topic 1).

2. **Q: How is a virtual address split under paging, and how is each part used?**
   A: Into a Virtual Page Number (VPN), used to look up the corresponding physical frame number in the page table, and an offset, which identifies the specific byte within that page and passes through unchanged into the final physical address.

3. **Q: Besides the physical frame number, what other information does a page table entry typically store?**
   A: A valid bit (whether this mapping represents an actually-used page at all), protection bits (read/write/execute permissions), and additional bits like present, reference, and dirty, used respectively by swapping (Module 10) and page-replacement policies (Module 11).

## Summary

- A page table is a per-process structure mapping every virtual page to its physical frame, used to translate every memory access.
- A virtual address splits into a VPN (used to look up the page table) and an offset (which passes through unchanged into the physical address).
- A page table entry stores much more than a frame number: a valid bit, protection bits, and (relevant later) present, reference, and dirty bits.
- The next topic quantifies a real cost this design introduces: how much memory a naive, single-level page table itself consumes, motivating Module 09's performance and space-saving refinements.
