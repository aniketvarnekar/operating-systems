# Swap Space and the Present Bit

## Learning Objectives

By the end of this section you should be able to:
- Explain what swap space is and why disk (not more RAM) is the natural place to put temporarily-evicted pages
- Explain what the present bit in a page table entry indicates
- Explain why this mechanism lets total virtual memory across all processes exceed physical RAM

## Prerequisites

- Module 08, Topic 2 (Page Tables and Page Table Entries)
- Module 06, Topic 1 (The Address Space Abstraction)

## Motivation

Every module since Module 06 has quietly assumed that every page a process might touch is already sitting in physical RAM, ready to be translated to a frame. This topic removes that assumption — and shows the two pieces of infrastructure (swap space, the present bit) that make it possible for a machine to support more total virtual memory across all its running processes than it has physical RAM to hold at once.

## Problem Statement

Physical RAM is a fixed, finite resource. But processes' address spaces (Module 06, Topic 1), taken together across everything running on a machine at once, can easily add up to far more virtual memory than the machine's actual physical RAM capacity. If a page currently isn't needed *right now* by the process that owns it, could it be temporarily moved somewhere else — freeing up its physical frame for something that's needed more urgently — and brought back later if and when it's actually accessed again?

## Concept

### Swap Space

> **Swap space** is a reserved region of disk (or other persistent, non-volatile storage) that the OS uses as an overflow area for pages that don't currently fit in physical RAM. A page temporarily moved out of RAM and into swap space is said to have been **swapped out** (or **paged out**); moving it back into RAM later is **swapping in** (or **paging in**).

Disk is the natural choice for this overflow area specifically because it's non-volatile (Module 01, Topic 5) and, critically, available in far greater capacity than RAM at a much lower cost per byte — trading access speed (disk is dramatically slower than RAM) for the ability to hold vastly more data than physical RAM ever could alone. This is the direct mechanism that lets the *sum* of every running process's active memory usage exceed the machine's actual physical RAM size — the difference is made up by swap space, holding whatever doesn't currently fit.

### The Present Bit

Recall from Module 08, Topic 2 that a page table entry (PTE) contains more than just a physical frame number — including a bit, previewed there, specifically for this module:

> The **present bit** in a page table entry indicates whether that virtual page is currently resident in physical RAM (present bit = 1, and the PTE's frame number is valid and usable for translation) or has been swapped out to disk (present bit = 0, meaning the frame number field is not a valid physical location at all — instead, the PTE typically repurposes that same space to record *where on disk* the page currently resides, within swap space).

This is a small but crucial distinction from the valid bit (also introduced in Module 08, Topic 2): the valid bit asks "does this virtual page correspond to any legitimate part of the address space at all?" (e.g., the unused heap/stack gap has invalid entries); the present bit asks a different question entirely, only meaningful for *valid* pages: "is this legitimate page currently sitting in RAM, or has it been temporarily relocated to disk?"

## Internal Working (Preview)

```
   Page Table Entry, PRESENT (in RAM):
   ┌───────┬────────┬──────────────┬───────────────┐
   │ Valid=1 │ Prot.    │ Present=1     │ Frame # = 47    │  ← normal translation
   └───────┴────────┴──────────────┴───────────────┘

   Page Table Entry, NOT PRESENT (swapped out to disk):
   ┌───────┬────────┬──────────────┬───────────────┐
   │ Valid=1 │ Prot.    │ Present=0     │ Disk location   │  ← NOT a physical frame;
   └───────┴────────┴──────────────┴───────────────┘     an access here must first
                                                            bring the page back into RAM
                                                            (Topic 2: the page fault)


   Physical RAM (limited)              Swap Space (on disk, larger, slower)
   ┌─────────────────┐                ┌─────────────────────────┐
   │ actively-used pages│  ◄── swap out ─│  temporarily evicted pages │
   │                     │  ── swap in ──►│                             │
   └─────────────────┘                └─────────────────────────┘
```

## Real-World Analogy

Think of physical RAM like the limited shelf space in a small retail store's front display area, and swap space like the store's much larger, but slower-to-access, back warehouse. Items selling actively (frequently accessed pages) stay out on the front shelves for quick, immediate customer access. Items that haven't sold in a while get moved to the back warehouse to free up front-shelf space for something more in-demand right now — the warehouse can hold vastly more total inventory than the front shelves ever could, at the cost of a real delay if a customer suddenly wants something currently sitting in the back (a page fault, Topic 2) and a staff member has to go retrieve it before it can be handed over.

## Why This Design Is Necessary

Without swap space, the total memory any process (or the sum of all running processes) could ever use would be hard-capped at the machine's exact physical RAM size — a severe, artificial limitation, given that most processes only actively use a small fraction of their allocated address space at any given moment (Module 06, Topic 1's heap/stack gap, once again). Allowing rarely-used pages to live temporarily on slower, more abundant disk storage — while keeping actively-used pages in fast RAM — lets the *illusion* of abundant memory (part of the transparency goal, Module 06, Topic 1) extend well beyond what physical RAM alone could ever provide.

## Advantages of Swap Space and the Present Bit

- **Total virtual memory can exceed physical RAM** — the single biggest practical benefit, letting a machine run more (or larger) processes than its physical RAM alone would otherwise allow.
- **Reuses existing page table infrastructure** — the present bit is simply one more field in the same PTE structure already established in Module 08, Topic 2, requiring no fundamentally new translation mechanism, just an additional check.

## Disadvantages / Costs

- **Disk access is dramatically slower than RAM access** — often by several orders of magnitude — so a page fault requiring a swap-in (Topic 2) is a genuinely expensive event, nothing like the cost of a normal memory access or even a TLB miss (Module 09, Topic 1).
- **Introduces the need for a replacement policy** — since swap space, while larger than RAM, is still finite, and more importantly, since physical RAM itself has a hard limit on how many pages it can hold present at once, the OS must decide *which* currently-present page to evict when a new page needs to come in and RAM is full — the subject of Module 11.

## Best Practices

- Keep the valid bit and present bit conceptually separate: valid asks "is this part of the address space in use at all," while present asks "is this in-use page currently in RAM or out on disk" — conflating them is a common source of confusion.
- When reasoning about a system's real-world performance under memory pressure, remember that swap space trades capacity for speed — relying on it heavily (rather than as an occasional overflow) can cause severe, very noticeable slowdowns, a phenomenon explored further in Module 11 (thrashing).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The present bit and the valid bit mean the same thing." | The valid bit indicates whether a virtual page corresponds to any legitimate, in-use part of the address space at all. The present bit, meaningful only for valid pages, indicates whether that legitimate page is currently resident in RAM or has been swapped out to disk. |
| "Swap space lets a single process use unlimited memory with no real cost." | Swap space is still finite, and more importantly, accessing a swapped-out page requires a genuinely expensive disk operation (Topic 2) to bring it back into RAM — relying on swapping heavily carries a real, often severe, performance cost. |

## Interview Questions

1. **Q: What is swap space, and why is disk used for it rather than just requiring more RAM?**
   A: Swap space is a reserved region of disk used as an overflow area for pages that don't currently fit in physical RAM. Disk is used because it's non-volatile and available in far greater capacity at much lower cost per byte than RAM, trading access speed for the ability to hold vastly more data than physical RAM alone could.

2. **Q: What does the present bit in a page table entry indicate, and how does it differ from the valid bit?**
   A: The present bit indicates whether a legitimate (valid) page is currently resident in physical RAM or has been swapped out to disk. The valid bit, by contrast, indicates whether a virtual page corresponds to any legitimate part of the address space at all, regardless of whether it's currently in RAM.

3. **Q: How does swap space let the total memory used by running processes exceed physical RAM?**
   A: Pages not currently needed can be temporarily moved out of RAM into swap space, freeing their physical frames for pages that are more urgently needed — the difference between total demanded memory and actual RAM capacity is made up by swap space holding whatever doesn't currently fit.

## Summary

- Swap space is a disk-based overflow area for pages that don't currently fit in physical RAM, trading disk's slower access speed for its much greater capacity.
- The present bit, a page table entry field, indicates whether a valid page is currently resident in RAM or has been swapped out — distinct from the valid bit, which indicates whether the page is part of the address space at all.
- Together, these let the total virtual memory in use across all processes exceed the machine's actual physical RAM.
- The next topic covers exactly what happens when a process accesses a page whose present bit is 0 — the page fault handling path.
