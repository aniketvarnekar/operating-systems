# Data Integrity via Checksums

## Learning Objectives

By the end of this section you should be able to:
- Explain what silent data corruption is, and why it's a distinct failure mode from both RAID's disk failures and crash consistency's mid-write interruptions
- Explain what a checksum is and how it detects (but does not, by itself, correct) corruption
- Explain how checksums combine with RAID-style redundancy to both detect and correct corruption

## Prerequisites

- Module 18 (RAID)
- Module 22 (Crash Consistency)

## Motivation

Modules 18 and 22 covered two specific, serious failure modes: an entire disk failing (RAID), and a crash interrupting a multi-step update (crash consistency). Neither addresses a third, quieter danger: a disk that appears to be working perfectly fine, and wasn't interrupted mid-write at all, can still occasionally return **subtly wrong data** for a read — silent corruption, with no error reported at all. This topic covers how systems detect this.

## Problem Statement

Physical storage media (Module 17) can, in rare cases, develop small defects that cause a stored bit to be read back incorrectly — not because the whole disk failed (Module 18's concern), and not because a write was ever interrupted (Module 22's concern), but simply because the physical media itself degraded slightly at one specific location. The disk hardware may report the read as entirely successful, since from its own perspective, it retrieved *something* without any I/O error at all — the data returned is just quietly wrong. How can a system detect this kind of corruption at all, given that nothing about the read operation itself signals a problem?

## Concept

### Silent Data Corruption

> **Silent data corruption** refers to data being altered or misread at the physical storage level, with the read operation itself reporting success — no I/O error, no exception, nothing to indicate anything went wrong — even though the actual bytes returned no longer match what was originally written.

This is a genuinely distinct failure mode from RAID's "a whole disk stopped responding" (Module 18) and crash consistency's "an update was interrupted partway through" (Module 22) — it can occur on a perfectly healthy-seeming disk, with no crash or hardware failure event involved at all, purely from rare physical media degradation.

### Checksums: Detecting Corruption

> A **checksum** is a small value computed from a larger block of data (using a mathematical function sensitive to virtually any change in the input), stored alongside that data. To verify the data's integrity later, the system recomputes the checksum from the data it actually reads back and compares it against the stored checksum — a mismatch reveals that the data has been altered since the checksum was originally computed.

```
   When WRITING:
     data = [ ... bytes ... ]
     checksum = compute_checksum(data)
     store BOTH data and checksum

   When READING (later):
     read data and its stored checksum
     recomputed_checksum = compute_checksum(the data just read)
     if recomputed_checksum != stored checksum:
         → CORRUPTION DETECTED (something silently changed)
```

Critically, a checksum by itself only **detects** that something has changed — it does not, on its own, tell you what the *original*, correct data actually was, or provide any way to recover it.

### Combining Checksums With Redundancy: Detection AND Correction

Detection alone isn't fully satisfying — ideally, a system would also be able to **correct** the corruption once detected, not just report that it happened. This is precisely where checksums combine powerfully with RAID-style redundancy (Module 18): if a checksum mismatch reveals that one disk's copy of a block is corrupted, and a mirrored copy (RAID 1) or a parity-reconstructable version (RAID 4/5) of that same data exists on another disk, the system can use that redundant copy to determine — and restore — the correct, original data, rather than merely reporting that something went wrong.

> This is the precise reason checksums and RAID-style redundancy are often deployed **together**: RAID alone can correct for an entire disk failing, but has no way to know a *specific block* silently returned wrong data from an otherwise-healthy disk; checksums alone can detect that a specific block is wrong, but have no way to recover the correct value on their own. Combined, checksums identify exactly *which* block is corrupted, and redundancy supplies the correct replacement.

## Internal Working (Preview)

```
   WITHOUT checksums:

     Disk returns: "here's your data" (data is actually SILENTLY WRONG)
     System has NO WAY to know anything is wrong at all


   WITH checksums (detection only):

     Disk returns data + stored checksum
     System recomputes checksum from the data → MISMATCH
     System now KNOWS this specific block is corrupted
     ... but doesn't know the correct value


   WITH checksums + RAID redundancy (detection AND correction):

     Checksum mismatch on Disk 1's copy of Block X
                    │
                    ▼
     System retrieves Block X's redundant copy/reconstruction
     from Disk 2 (RAID 1 mirror) or via parity (RAID 4/5)
                    │
                    ▼
     Verifies THAT copy's checksum too → if it matches, USE IT
     as the correct value, and optionally repair Disk 1's copy
```

## Real-World Analogy

Think of a checksum like a tamper-evident seal on a package: the seal itself doesn't tell you what the package's *original* correct contents were, but a broken seal tells you, unambiguously, that something has changed since it was sealed — exactly analogous to a checksum mismatch. If you also happen to have a second, identically-sealed backup package stored somewhere else (RAID-style redundancy), a broken seal on the first package tells you specifically *not* to trust its contents, and you can instead open the second, still-sealed package to get the correct, original contents — detection (the broken seal) combined with redundancy (the backup package) gives you both "something's wrong" and "here's how to fix it."

## Why This Design Is Necessary

RAID (Module 18) is specifically designed around the failure model of an entire disk becoming unavailable — it has no mechanism to notice that a specific, individual block on an otherwise fully-functioning disk has silently returned incorrect data. Crash consistency (Module 22) is specifically designed around the failure model of an interrupted multi-step update — it has nothing to say about data that was written correctly, fully completed, and only later became corrupted due to physical media degradation. Checksums fill precisely this gap: a lightweight, general mechanism for detecting *any* unexpected change to data, regardless of cause, which then lets existing redundancy mechanisms (Module 18) be leveraged for correction once a problem is actually identified.

## Advantages of Checksums

- **Detects a failure mode neither RAID nor crash consistency addresses at all** — silent corruption on an otherwise healthy, uninterrupted disk.
- **Lightweight** — computing and storing a small checksum alongside data is a modest overhead compared to the protection it provides.
- **Combines naturally with existing redundancy** (RAID, Module 18) to provide both detection and correction, rather than requiring an entirely separate correction mechanism.

## Disadvantages / Limitations

- **Checksums alone only detect corruption — they cannot correct it** without some form of redundancy to supply the correct replacement value.
- **Storage and computation overhead** — every block of data requires its checksum to be computed (on write) and verified (on read), a real, if usually small, cost applied to every single piece of protected data.

## Best Practices

- Recognize checksums and RAID as solving genuinely different problems (silent corruption vs. entire disk failure) — deploying only one leaves a real gap the other is specifically designed to fill.
- When designing or evaluating a storage system's reliability story, explicitly ask: does it protect against entire disk failure (RAID)? Crash-mid-write inconsistency (journaling)? Silent per-block corruption (checksums)? A robust system typically needs all three, since each addresses a distinct failure mode.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "RAID protects against all forms of data corruption, including silent, single-block corruption on an otherwise healthy disk." | RAID's redundancy is designed around entire-disk failure; it has no built-in mechanism to detect that a specific block silently returned incorrect data from a disk that otherwise appears to be working fine — that's precisely the gap checksums fill. |
| "A checksum mismatch tells the system what the correct data actually was." | A checksum only detects that something has changed since it was computed — recovering the actual correct value requires some separate source of redundancy (like a RAID mirror or parity reconstruction), which the checksum itself does not provide. |

## Interview Questions

1. **Q: What is silent data corruption, and why is it a distinct failure mode from RAID's disk-failure protection?**
   A: Data being altered or misread at the physical level while the read operation itself reports success, with no I/O error at all. It's distinct from RAID's concern because it can occur on an otherwise fully healthy, functioning disk — RAID's redundancy is designed around an entire disk becoming unavailable, not a specific block silently returning wrong data.

2. **Q: What does a checksum do, and what is its key limitation on its own?**
   A: A checksum is a small value computed from a data block, stored alongside it, and recomputed and compared on later reads to detect whether the data has changed. On its own, it can only detect that corruption occurred — it provides no way to recover the original, correct data.

3. **Q: How do checksums combine with RAID to provide both detection and correction?**
   A: A checksum mismatch identifies exactly which specific block has been corrupted; RAID-style redundancy (a mirror copy or parity reconstruction) then supplies the correct replacement value for that identified block — detection and correction working together, each solving the part the other cannot.

## Summary

- Silent data corruption is data being altered at the physical level with no error reported by the read operation itself — a distinct failure mode from RAID's disk failures and crash consistency's mid-write interruptions.
- A checksum, computed and stored alongside data, detects corruption by comparing a recomputed value against the stored one on a later read — but cannot, by itself, recover the correct original value.
- Combining checksums with RAID-style redundancy provides both detection (identifying which block is wrong) and correction (supplying the correct replacement from redundant data).
- The next topic shifts from single-machine integrity concerns to networked file systems — accessing files that live on a completely different, physically remote machine.
