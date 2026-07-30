# Module 15 — Concurrency Bugs

## Module Goal

By the end of this module, you will understand **the two major categories of real-world concurrency bugs** — non-deadlock bugs (atomicity and order violations) and deadlock — including deadlock's exact required conditions, and the general strategies (prevention, avoidance, detection) for handling it, building directly on Module 14, Topic 3's concrete preview.

## Topics Covered in This Module

1. **[Non-Deadlock Bugs](01-non-deadlock-bugs.md)** — Atomicity violations and order violations: concurrency bugs that don't involve a waiting cycle at all.
2. **[Deadlock: Conditions and Prevention](02-deadlock-conditions-and-prevention.md)** — The four necessary conditions for deadlock, and prevention strategies that each attack one condition directly.
3. **[Deadlock Avoidance and Detection](03-deadlock-avoidance-and-detection.md)** — Proactively avoiding unsafe states versus letting deadlock happen and recovering from it.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 14 in full, especially Topic 3 (Classic Concurrency Problems) — this module formalizes what that topic demonstrated concretely.
- Module 13 in full.

## How to Study This Module

Read in order. Topic 1 is a useful corrective: many real-world concurrency bugs have nothing to do with deadlock at all — they're simpler, and arguably more common, mistakes in applying locks correctly. Topic 2 formalizes Module 14, Topic 3's concrete deadlock example into the four necessary conditions, and shows how each classic prevention strategy works by denying exactly one of them. Topic 3 covers the two remaining general strategies — avoidance (proactively refusing to enter a dangerous state) and detection (letting deadlock happen, but recognizing and recovering from it) — completing the standard four-strategy toolkit (prevention, avoidance, detection, and — implicitly — acceptance, when none of the others is practical).
