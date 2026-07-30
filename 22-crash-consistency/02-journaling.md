# Journaling

## Learning Objectives

By the end of this section you should be able to:
- Explain the core idea of write-ahead logging
- Walk through the journaling sequence: journal write, commit, checkout/checkpoint
- Explain why journaling only needs to replay the journal after a crash, not scan the entire disk

## Prerequisites

- Topic 1 (The Crash Consistency Problem and fsck)
- Module 01, Topic 5 (Persistence) — specifically, the journal-first analogy previewed there

## Motivation

Topic 1 showed fsck's fundamental limitation: it must scan the *entire* disk after any unclean shutdown, regardless of how small the interrupted operation actually was. This topic covers journaling — the technique, previewed conceptually all the way back in Module 01, Topic 5's accounting-ledger analogy, that avoids this problem entirely by changing *how* writes happen, not just how they're checked afterward.

## Problem Statement

Instead of writing directly to a file system's actual, permanent structures (inodes, bitmaps, directories) and hoping nothing gets interrupted partway through — then needing a full-disk scan (Topic 1's fsck) to clean up if it does — could a file system record its **intent** somewhere safe *first*, in a way that makes recovery from any interruption fast and precise, rather than requiring an exhaustive, disk-wide search?

## Concept

### Write-Ahead Logging: The Core Idea

> **Journaling** writes a description of an intended update — every change about to be made to the file system's permanent structures — to a dedicated, sequential **journal** (or log) area **before** actually applying those changes to their real, final locations on disk. If a crash occurs before the real update is fully applied, the journal itself contains everything needed to either finish the update correctly or safely discard it.

This is the exact mechanism Module 01, Topic 5 introduced via its accounting-ledger analogy: write down "I am about to change entry #42 from $100 to $150" in a journal *before* actually editing the master ledger — if interrupted, anyone can read the journal afterward and know exactly how to safely complete (or safely ignore) that specific pending change.

### The Journaling Sequence

A typical journaled update follows a specific, carefully ordered sequence:

1. **Journal write**: write a complete description of the intended changes (e.g., "update inode #500 to X, add directory entry Y") to the journal — this is itself a write to disk, but crucially, it's a small, separate area, and doesn't touch any of the file system's actual permanent structures yet.
2. **Commit**: write a special "commit" marker to the journal, indicating that the *entire* intended update has been fully and safely recorded in the journal — this is the critical point that separates "we can safely proceed" from "we can safely discard this attempt entirely."
3. **Checkout (checkpoint)**: **now** actually apply the described changes to the file system's real, permanent structures (the actual inodes, bitmaps, directories).
4. **Free**: once the checkpoint is confirmed complete, the corresponding journal entry can be marked as no longer needed (its job is done).

### Why This Makes Crash Recovery Fast and Precise

If a crash occurs **before** the commit marker is written, the journal entry is **incomplete** — the recovery process simply discards it entirely (as if the operation never started at all), since none of the actual permanent structures were touched yet (step 3 hadn't begun).

If a crash occurs **after** the commit marker but **before** the checkout finishes, the journal entry is **complete and committed** — the recovery process simply **replays** it: re-applies the exact same described changes to the permanent structures, finishing the checkout step that was interrupted. Since the journal entry fully describes the intended changes, replaying it produces the correct, consistent final result, regardless of exactly how far the original checkout had gotten before the crash.

> Critically, recovery only ever needs to examine the **journal** itself — a small, dedicated area — never the entire disk. This is precisely what avoids fsck's full-disk-scan cost (Topic 1): the journal directly tells recovery exactly what might be incomplete, without needing to exhaustively cross-check every structure on the whole disk to find out.

## Internal Working (Preview)

```
   JOURNALED UPDATE SEQUENCE:

   1. JOURNAL WRITE: "intend to: update inode #500, add dir entry Y"
                       (written to the JOURNAL area, not yet applied
                        to real structures)
   2. COMMIT:          write a COMMIT marker for this journal entry
   3. CHECKOUT:         NOW apply the actual changes to inode #500
                        and the real directory structure
   4. FREE:             mark this journal entry as done, reusable


   CRASH SCENARIOS AND RECOVERY:

   Crash BEFORE commit:
     journal entry INCOMPLETE → recovery DISCARDS it entirely
     (real structures were never touched — nothing to undo)

   Crash AFTER commit, DURING/BEFORE checkout:
     journal entry COMPLETE & COMMITTED → recovery REPLAYS it
     (re-applies the exact described changes — finishes what
      the checkout was interrupted mid-way through)

   Recovery ONLY reads the (small) JOURNAL — never scans the
   entire disk the way fsck (Topic 1) must
```

## Real-World Analogy

This is precisely Module 01, Topic 5's accounting analogy, made fully concrete: before actually editing the master ledger (the file system's real structures), the bookkeeper first writes a clear note in a separate journal: "About to change entry #42 from $100 to $150" (the journal write), then puts a checkmark next to that note once they're confident it's fully and correctly written down (the commit), and *only then* actually goes and edits the master ledger itself (the checkout). If the bookkeeper is interrupted at any point, anyone can look at the journal: an unchecked note means the change never really started (safely ignore it); a checked note means the intended change is fully known and can simply be carried out from the note, exactly as written, regardless of how far the actual ledger-editing had gotten.

## Why This Design Is Necessary

Fsck's approach (Topic 1) treats every crash as a mystery requiring an exhaustive, whole-disk investigation to solve. Journaling instead makes the file system's own *intent* explicit and durable *before* acting on it — turning "what might have gone wrong, somewhere on this entire disk?" into "what does the journal say was in progress, specifically?" This is a direct, general instance of a powerful pattern: recording intent durably before acting is what allows any interruption to be resolved precisely, without needing to reconstruct intent after the fact through broad, expensive inspection.

## Advantages of Journaling

- **Fast recovery** — only the (small) journal needs to be examined after a crash, not the entire disk, directly solving fsck's scaling problem.
- **Precise, guaranteed-correct recovery** — replaying a committed journal entry always produces a fully correct, consistent result, since the entry fully describes the intended change.

## Disadvantages / Costs

- **Every update now involves writing the data twice** — once to the journal, once to its final location (the checkout) — a real, ongoing overhead in exchange for fast, reliable crash recovery.
- **The journal area itself consumes some dedicated disk space** and requires its own management (reusing space once entries are freed, Topic 2's step 4).

## Best Practices

- When explaining why modern systems recover from an improper shutdown almost instantly, compared to older systems that could take a very long time, attribute the difference directly to journaling versus fsck's full-disk-scan approach.
- Recognize the "write intent first, durably, before acting" pattern as broadly reusable beyond file systems specifically — it's the same underlying idea behind many other crash-safe or fault-tolerant system designs.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Journaling means writing data to disk only once, just in a different order." | Journaling requires writing the intended changes twice — once to the journal (recording intent) and once to the actual permanent structures (the checkout) — genuinely more total disk writes than a non-journaled approach, in exchange for reliable, fast recovery. |
| "After a crash, a journaling file system still needs to scan the entire disk to find what went wrong, just like fsck." | Journaling recovery only needs to examine the journal itself — a small, dedicated area — since the journal directly and precisely records exactly what update was in progress, avoiding the need for an exhaustive, disk-wide consistency check. |

## Interview Questions

1. **Q: What is the core idea behind journaling (write-ahead logging)?**
   A: Writing a complete description of an intended update to a dedicated journal area before applying those changes to the file system's real, permanent structures — so that any crash can be resolved by consulting the journal alone, rather than needing to inspect the entire disk.

2. **Q: What are the main steps in a journaled update, and why does the commit step matter?**
   A: Journal write (record the intended changes), commit (mark the journal entry as fully and safely recorded), checkout (apply the changes to the real structures), and free (mark the entry as done). The commit step is the critical dividing line: before it, a crash means the journal entry is incomplete and can be safely discarded; after it, the entry is guaranteed complete and can be safely replayed.

3. **Q: Why does journaling avoid the full-disk-scan cost that fsck requires?**
   A: Because the journal itself precisely records exactly what update was in progress at the time of a crash, recovery only needs to examine that small, dedicated journal area — replaying any committed-but-not-yet-checked-out entries — rather than exhaustively cross-checking every structure on the entire disk to detect what might be wrong.

## Summary

- Journaling writes a description of an intended update to a dedicated journal area before applying it to the file system's real structures, directly implementing Module 01, Topic 5's journal-first analogy.
- The commit marker is the critical dividing line: before it, a crash means the update is safely discardable; after it, the update is guaranteed completable by replaying the journal entry.
- Recovery only ever needs to examine the (small) journal, not the entire disk, making journaling dramatically faster to recover from than fsck's exhaustive full-disk scan.
- This closes out the module's coverage of crash consistency — the module summary ties the crash consistency problem, fsck, and journaling together before Module 23 covers data integrity via checksums and an introduction to distributed/networked file systems.
