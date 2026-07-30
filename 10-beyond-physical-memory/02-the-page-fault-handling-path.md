# The Page Fault Handling Path

## Learning Objectives

By the end of this section you should be able to:
- Define a page fault precisely, and distinguish it from an illegal memory access
- Walk through the complete page-fault handling sequence, step by step
- Explain why a page fault is handled via the same trap mechanism as a system call

## Prerequisites

- Topic 1 (Swap Space and the Present Bit)
- Module 03, Topic 2 (Restricted Operations and Traps)

## Motivation

Topic 1 introduced the present bit as the signal that a page has been swapped out. This topic covers what actually happens the instant a process tries to access such a page — the mechanism that makes swap space (Topic 1) actually usable, rather than just a place pages sit inertly and unreachably once evicted.

## Problem Statement

A process executes an ordinary instruction that references a virtual address — no different, from the process's own perspective, than any other memory access. But the page containing that address currently has its present bit set to 0 (Topic 1) — it's been swapped out to disk. The hardware cannot simply complete a normal translation, since there is no valid physical frame number to use. What happens instead?

## Concept

### Definition

> A **page fault** is a hardware-detected exception, triggered when a process accesses a page whose present bit is 0 (Topic 1) — that is, a page that is valid (part of the address space) but not currently resident in physical RAM. The hardware traps into the OS (using the exact same trap mechanism from Module 03, Topic 2) to handle it.

This is worth pausing on: a page fault is not an error in the sense of a program doing something illegal — it's an entirely expected, routine event that the OS is specifically designed to handle transparently, as part of making swap space (Topic 1) actually work. This is distinct from an **illegal access** (referencing a page whose valid bit is 0, or attempting an operation the protection bits forbid — Module 08, Topic 2), which the OS typically handles by terminating the offending process instead, since there's no legitimate page to bring in at all.

### The Page-Fault Handling Sequence

1. A process's instruction generates a virtual address whose corresponding page table entry has present = 0.
2. The hardware detects this during translation and raises a page-fault exception — a trap, using the exact same trap-table mechanism (Module 03, Topic 2) that handles system calls and other exceptions, jumping to a specific, pre-registered kernel handler for page faults.
3. The kernel's page-fault handler examines the faulting page table entry to determine where on disk (within swap space, Topic 1) this page's data currently resides.
4. The kernel checks whether a free physical frame is currently available. **If physical RAM is completely full**, the kernel must first choose an existing resident page to evict, making room — this decision is a page-replacement **policy** question, covered fully in Module 11; this module covers only the mechanism of bringing a page in, not which page to evict to make room for it.
5. The kernel issues a request to read the faulting page's data from swap space into the (now available) physical frame — this is a genuine disk I/O operation, and the process is moved to the Blocked state (Module 02, Topic 2) while it completes, exactly like any other process waiting on slow I/O.
6. Once the disk read completes, the kernel updates the page table entry: present is set back to 1, and the frame number field is updated to the newly-loaded physical frame.
7. The kernel resumes the faulting process — typically by re-executing the exact same instruction that originally caused the fault. This time, translation succeeds normally, since the page is now present, and the process continues as if nothing unusual happened at all (directly fulfilling the transparency goal, Module 06, Topic 1).

## Internal Working (Preview)

```
   Process accesses virtual address in a page with present = 0
              │
              ▼
   HARDWARE detects this during translation → raises a page fault
   (a TRAP — same mechanism as Module 03, Topic 2's trap table)
              │
              ▼
   KERNEL page-fault handler runs:
     1. locate the page's data in swap space (Topic 1)
     2. is a free physical frame available?
              │
        ┌─────┴─────┐
        ▼             ▼
       YES            NO → choose a page to evict
        │                  (POLICY — Module 11, not this module)
        │                       │
        └───────────┬───────────┘
                     ▼
     3. read the page's data from disk into the frame
        (process is BLOCKED during this real disk I/O —
         Module 02, Topic 2)
                     ▼
     4. update the PTE: present = 1, frame # = new frame
                     ▼
     5. RESUME the process — re-execute the SAME faulting
        instruction; this time, translation succeeds normally
```

## Real-World Analogy

Recall Topic 1's warehouse analogy: a page fault is exactly like a customer asking for an item that's currently stored in the back warehouse rather than out on the front shelf. The clerk (the hardware) recognizes immediately that this specific item isn't on the shelf, and instead of simply failing the request, discreetly signals a stockroom associate (the kernel's page-fault handler) to go retrieve it. If the front shelf happens to be completely full already, the associate first has to decide which existing item to move to the back to make room (Module 11's policy question) before fetching the requested item. The customer waits (the process is Blocked) while this happens, and once the item is placed on the shelf, the *original* request is fulfilled exactly as if it had been on the shelf all along — the customer never has to re-explain or reformulate their request from scratch.

## Why Handling a Page Fault as a Trap Makes Sense

A page fault needs exactly the same properties Module 03, Topic 2 established for traps generally: it must safely transfer control to trusted kernel code, at a fixed, pre-registered entry point, with no way for a user process to redirect it elsewhere or bypass it. Reusing the existing trap-table infrastructure (rather than inventing an entirely separate mechanism) is a direct, elegant application of a general-purpose tool (traps) to a new specific problem — precisely the kind of reuse Module 02, Topic 7 highlighted as a recurring, celebrated pattern in OS design (there, fork/exec's composability; here, the trap mechanism's generality).

## Advantages of This Design

- **Complete transparency to the faulting process** — by re-executing the exact same instruction after the fault is resolved, the process has no way to detect that a page fault (and a real disk operation) occurred at all, directly fulfilling Module 06, Topic 1's transparency goal.
- **Reuses existing, well-understood infrastructure** — the same trap mechanism (Module 03, Topic 2) that handles system calls handles page faults, requiring no separate control-transfer mechanism.
- **Lets the CPU stay productive during the disk wait** — because the faulting process is moved to Blocked (Module 02, Topic 2) rather than the CPU idling, the scheduler (Modules 04–05) can run a different ready process during the (comparatively very long) disk read, directly applying the overlap principle from Module 04, Topic 6.

## Disadvantages / Costs

- **A page fault requiring a real disk read is extremely expensive** relative to a normal memory access or even a TLB miss (Module 09, Topic 1) — disk latency is vastly higher than RAM latency, making page faults something a well-tuned system tries hard to minimize in frequency (directly motivating Module 11's replacement policies, which aim to keep the *right* pages resident).
- **A page fault occurring when RAM is completely full requires an eviction decision first** — adding the cost and complexity of a full page-replacement policy (Module 11) on top of the disk read itself.

## Best Practices

- Always distinguish a page fault (a valid page, temporarily not present — handled transparently) from an illegal access (an invalid page, or a disallowed operation — typically resulting in process termination) — conflating the two is a common source of confusion.
- When reasoning about why a system feels sluggish under heavy memory pressure, connect it directly to a high rate of expensive page faults requiring real disk I/O — this is the mechanical root cause Module 11 explores further (thrashing).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A page fault always means something has gone wrong in the program." | A page fault for a valid, merely-swapped-out page is a routine, expected event the OS handles transparently — it's fundamentally different from an illegal access (an invalid page or forbidden operation), which is the case that typically results in the process being terminated. |
| "After a page fault is resolved, the process continues from the next instruction, having lost the one that faulted." | The kernel re-executes the exact same instruction that originally caused the fault, now that the page is present — nothing is skipped or lost; from the process's perspective, that instruction simply took a bit longer than usual to complete. |

## Interview Questions

1. **Q: What is a page fault, and how does it differ from an illegal memory access?**
   A: A page fault is a hardware-detected exception triggered when a process accesses a valid page that isn't currently resident in RAM (present bit = 0) — a routine, expected event the OS handles by bringing the page in from swap space. An illegal access (an invalid page or a disallowed operation per the protection bits) has no legitimate page to bring in, and typically results in the offending process being terminated instead.

2. **Q: Walk through what happens after a page fault is triggered.**
   A: The hardware traps into the kernel's page-fault handler, which locates the page's data in swap space, ensures a free physical frame is available (evicting an existing page first if RAM is full — a Module 11 policy decision), issues a disk read to bring the page's data into that frame while the process is Blocked, updates the page table entry (present = 1, new frame number), and then resumes the process by re-executing the original faulting instruction.

3. **Q: Why is a page fault handled using the same trap mechanism as a system call?**
   A: Because it needs the same properties: a safe, hardware-enforced transfer of control to trusted kernel code at a fixed, pre-registered entry point. Reusing the existing trap-table infrastructure (Module 03, Topic 2) avoids inventing a separate control-transfer mechanism for what is, mechanically, just another kind of trap.

## Summary

- A page fault occurs when a process accesses a valid page whose present bit is 0, triggering a hardware trap into the kernel — a routine, expected event, not an error.
- The kernel locates the page in swap space, ensures a free frame is available (evicting an existing page if needed — Module 11's policy question), reads the page in from disk while the process is Blocked, updates the page table entry, and resumes the process by re-executing the original faulting instruction.
- This reuses the exact same trap mechanism established in Module 03, Topic 2, applying it to a new kind of event.
- This closes out the module's mechanism-focused coverage of memory beyond physical RAM — the module summary ties swap space, the present bit, and the fault-handling path together before Module 11 addresses the policy question this topic deliberately deferred: which page to evict when physical memory is full.
