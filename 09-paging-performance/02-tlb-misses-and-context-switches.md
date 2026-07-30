# TLB Misses and Context Switches

## Learning Objectives

By the end of this section you should be able to:
- Explain why a TLB entry that was correct for one process becomes dangerous for a different process
- Compare TLB flushing and address-space identifiers (ASIDs) as two solutions to this problem
- Explain the performance trade-off each solution makes

## Prerequisites

- Topic 1 (The Translation Lookaside Buffer)
- Module 03, Topic 4 (The Context Switch)

## Motivation

Topic 1 established that the TLB caches recent translations for speed. But every one of those cached translations is specific to whichever process generated it — Process A's virtual page 5 maps to a completely different physical frame than Process B's virtual page 5 (each process has its own independent page table, Module 08, Topic 2). This topic covers the real danger this creates the moment a context switch (Module 03, Topic 4) happens, and the two classic fixes.

## Problem Statement

Suppose Process A is running, and its virtual page 5 has been recently translated and cached in the TLB, pointing to physical frame 20. The OS then performs a context switch (Module 03, Topic 4) to Process B. Process B *also* has a virtual page 5 in its own address space — but Process B's virtual page 5 is supposed to map to a completely different physical frame, say frame 47, according to *Process B's own* page table (Module 08, Topic 2). If the TLB still contains the old, stale entry from Process A ("virtual page 5 → physical frame 20") and nothing is done about it, Process B's access to its own virtual page 5 could incorrectly reuse Process A's cached translation — reading or writing Process A's physical memory instead of its own. This isn't just an efficiency problem; it's a serious protection failure, a direct violation of the isolation goal (Module 06, Topic 1).

## Concept

### Solution 1: TLB Flushing

> **TLB flushing** clears (invalidates) all entries in the TLB every time a context switch occurs, guaranteeing that the newly-scheduled process starts with no stale translations from a different process to accidentally reuse.

This is simple and completely correct — it fully eliminates the cross-process staleness danger from the Problem Statement, by ensuring no old process's entries are ever still present when a different process resumes. But it comes with a real performance cost: since the TLB starts empty after every single context switch, the newly-scheduled process must pay the full TLB-miss cost (Topic 1) for every one of its distinct pages all over again, until its own working set of translations is gradually re-cached — losing all the benefit locality would otherwise provide, precisely at the moments (right after a switch) when a fresh burst of accesses is about to happen.

### Solution 2: Address Space Identifiers (ASIDs)

> An **Address Space Identifier (ASID)** is a small tag, stored alongside each TLB entry, identifying *which process* that specific translation belongs to. Instead of clearing the TLB on every context switch, the hardware simply includes the current process's ASID as part of every TLB lookup — an entry only counts as a match (a hit) if both its VPN *and* its ASID match the currently running process.

This allows multiple processes' translations to safely coexist in the TLB simultaneously, without ever being confused for each other, since a lookup for Process B's virtual page 5, tagged with Process B's ASID, simply will not match Process A's cached entry for virtual page 5, tagged with Process A's ASID — even though the raw VPN number is identical in both. This preserves useful cached translations across context switches (avoiding TLB flushing's full reset), at the modest hardware cost of a few extra tag bits stored per TLB entry, and the OS's responsibility to assign and track a distinct ASID per process.

## Internal Working (Preview)

```
 Without ASIDs — TLB flushing required:

   Process A running: TLB has [VPN 5 → Frame 20]
   Context switch to Process B  ──►  FLUSH: clear entire TLB
   Process B running: TLB is empty — every access is a miss at first,
                       even for pages B has accessed many times before


 With ASIDs — no flush needed:

   Process A running (ASID=1): TLB has [ASID=1, VPN 5 → Frame 20]
   Context switch to Process B (ASID=2)  ──►  NO FLUSH
   Process B running: lookup for [ASID=2, VPN 5] does NOT match the
                       existing [ASID=1, VPN 5] entry — correctly
                       treated as a miss, safely, without confusing
                       the two processes' translations
   (Process A's entry remains cached, ready for a hit
    the next time Process A is scheduled again)
```

## Real-World Analogy

Recall Topic 1's spice-rack analogy. TLB flushing is like a shared kitchen countertop that gets completely wiped clean of all spices every single time a *different* chef starts their shift — perfectly safe (no chef ever mistakes another's ingredients for their own), but wasteful, since every chef has to re-gather their commonly-used spices from the pantry all over again at the start of every single shift, even spices they set out just a few minutes earlier. ASIDs are like giving every chef's spice jars a small, distinct colored label — all the jars can stay out on the shared countertop simultaneously, and each chef simply only reaches for jars with their own color, safely ignoring (never confusing) another chef's identically-named ingredient sitting right next to it.

## Why This Trade-off Exists

TLB flushing prioritizes simplicity and guaranteed correctness at the cost of significant performance loss right after every context switch — losing exactly the cached-translation benefit the TLB exists to provide. ASIDs prioritize performance (preserving useful cached entries across switches) at the cost of slightly more complex hardware (an extra tag field per entry, and comparison logic that checks both VPN and ASID) and OS-level bookkeeping (assigning, tracking, and eventually reusing a limited supply of ASID values across all running processes). Given how frequently context switches occur (Modules 03–05), real-world systems overwhelmingly favor ASID-based designs specifically to avoid paying TLB flushing's performance cost on every single switch.

## Advantages of ASIDs Over Flushing

- **Preserves cached translations across context switches** — a process resumed after being switched out can still benefit from its own previously-cached TLB entries, rather than starting cold every time.
- **Better performance under frequent context switching** — since scheduling policies (Modules 04–05) can switch processes quite often (e.g., Round Robin's short time slices), avoiding a full TLB flush on every single switch meaningfully reduces overall TLB-miss overhead.

## Disadvantages / Trade-offs

- **Additional hardware complexity** — every TLB entry needs extra bits for the ASID tag, and lookup logic must compare both VPN and ASID together.
- **Limited number of available ASIDs** — since ASID tag fields are a fixed, finite size, only a limited number of distinct ASIDs can exist at once; the OS must manage reuse (and, when values must be recycled among many active processes, may still need to fall back to flushing in some cases).

## Best Practices

- When explaining TLB behavior across context switches, always ask "flush or ASID-tagged?" as an explicit design question — the two approaches make a genuine correctness-vs-performance trade-off, not merely a stylistic implementation detail.
- Recognize that ASIDs are conceptually similar to Module 08, Topic 2's per-process page tables: both exist because the *same* virtual address (or VPN) means something entirely different depending on *which process* is asking — ASIDs simply extend that same per-process distinction into the TLB's caching layer.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The TLB automatically knows which process a cached entry belongs to, without any extra information." | Without an explicit mechanism (an ASID tag, or a full flush on switch), a cached VPN-to-frame mapping has no inherent way to indicate which process it belongs to — reusing it for the wrong process would silently and dangerously misdirect a memory access. |
| "TLB flushing is a performance bug in real systems, since ASIDs are strictly better." | Flushing is simpler and requires no extra hardware support; on hardware that doesn't implement ASIDs at all, flushing remains the only correct option, despite its performance cost. ASIDs are an optimization available only when the hardware explicitly supports them. |

## Interview Questions

1. **Q: Why is it dangerous to leave TLB entries unchanged across a context switch?**
   A: Because a TLB entry's VPN-to-physical-frame mapping is specific to the process that generated it; reusing it for a different process (whose same VPN number maps to an entirely different physical frame in its own page table) would incorrectly redirect that process's memory access, violating process isolation.

2. **Q: What are the two classic solutions to this problem, and what's the trade-off between them?**
   A: TLB flushing clears the entire TLB on every context switch — simple and always correct, but discards all cached translations, forcing the newly-scheduled process to re-populate them from scratch. ASIDs tag each TLB entry with the owning process's identifier, letting multiple processes' translations safely coexist in the TLB — better performance, at the cost of extra hardware support and OS bookkeeping.

3. **Q: How does an ASID prevent two different processes' identical VPNs from being confused in the TLB?**
   A: A TLB lookup only counts as a hit if both the VPN and the ASID match the currently running process — so Process B's lookup for its own virtual page 5, tagged with Process B's ASID, will not match Process A's cached entry for the identically-numbered virtual page 5, tagged with Process A's different ASID.

## Summary

- A TLB entry is specific to the process that generated it; reusing it after a context switch to a different process is a real correctness and isolation danger, not just a performance concern.
- TLB flushing clears the entire TLB on every context switch — simple and correct, but discards all useful cached translations every time.
- ASIDs tag each TLB entry with its owning process, letting multiple processes' entries safely coexist and avoiding the cost of a full flush, at the price of extra hardware support and OS bookkeeping.
- Having addressed the TLB's time-cost side of Module 08, Topic 3's cliffhanger, the next topic pivots to the space-cost side: multi-level page tables, which avoid reserving table space for entirely unused regions of an address space.
