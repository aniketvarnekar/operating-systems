# Deadlock: Conditions and Prevention

## Learning Objectives

By the end of this section you should be able to:
- State the four necessary conditions for deadlock precisely
- Map Module 14, Topic 3's dining philosophers example onto each of the four conditions
- Explain at least two prevention strategies, and which specific condition each one denies

## Prerequisites

- Module 14, Topic 3 (Classic Concurrency Problems)

## Motivation

Module 14, Topic 3 showed you a deadlock concretely and fixed it with one specific trick (breaking ordering symmetry). This topic generalizes that experience into a precise, formal statement of exactly what conditions must all hold simultaneously for deadlock to occur — and shows that every classic prevention strategy works by denying exactly one of those conditions.

## Problem Statement

Module 14, Topic 3's dining philosophers deadlock felt specific to forks and philosophers. Is there a more general, precise statement of exactly what has to be true for *any* deadlock — across any resources, any number of threads — to occur? And if so, could a system guarantee deadlock never happens simply by making sure at least one of those requirements is never satisfied?

## Concept

### The Four Necessary Conditions for Deadlock

> Deadlock can occur only if **all four** of the following conditions hold simultaneously:
>
> 1. **Mutual exclusion**: at least one resource must be held in a non-shareable way — only one thread can use it at a time (exactly like a lock, Module 13).
> 2. **Hold-and-wait**: a thread holding at least one resource is waiting to acquire additional resources currently held by other threads.
> 3. **No preemption**: resources cannot be forcibly taken away from a thread holding them — they can only be released voluntarily by the thread that holds them.
> 4. **Circular wait**: there exists a cycle of threads, each holding a resource that the next thread in the cycle is waiting for.

These are **necessary**, not merely sufficient in isolation — deadlock requires **all four** to hold at once; denying **any single one** of them makes deadlock impossible, which is exactly the basis for every prevention strategy below.

### Mapping This Onto Dining Philosophers

Recall Module 14, Topic 3's naive solution:

1. **Mutual exclusion**: each fork can only be held by one philosopher at a time. ✓
2. **Hold-and-wait**: each philosopher holds their left fork while waiting for their right fork. ✓
3. **No preemption**: a philosopher holding a fork will not be forced to give it up. ✓
4. **Circular wait**: philosopher 1 waits on philosopher 2's fork, philosopher 2 waits on philosopher 3's fork, and so on, all the way back around to philosopher 1 — a complete cycle. ✓

All four conditions held simultaneously — which is exactly why deadlock occurred. Module 14, Topic 3's fix (having one philosopher acquire forks in the opposite order) specifically **denies condition 4 (circular wait)**: with one philosopher's acquisition order reversed, no complete cycle of "everyone waiting on the next" can form.

### Prevention Strategies: Denying One Condition at a Time

Since all four conditions are necessary, a system can prevent deadlock entirely by guaranteeing that at least one of them can never hold:

- **Attacking mutual exclusion**: where possible, use resources that don't require exclusive access at all (e.g., a data structure that can safely be read by multiple threads simultaneously without a lock). Not always feasible — some resources (like a fork, or a genuinely exclusive hardware device) are inherently non-shareable.
- **Attacking hold-and-wait**: require a thread to acquire **all** the resources it will ever need at once, atomically, before starting — never holding some while waiting for others. If it can't get all of them immediately, it acquires none of them, and retries later. This denies hold-and-wait directly, since a thread is never simultaneously "holding one, waiting for another."
- **Attacking no preemption**: allow resources to be forcibly taken away from a thread that's been waiting too long (with the thread's own progress rolled back and later retried) — denying the "no preemption" condition, at the cost of added complexity in safely rolling back partial progress.
- **Attacking circular wait**: impose a **global, consistent ordering** on all resources, and require every thread to acquire resources strictly in that order, regardless of which specific resources it needs — this is precisely the general version of Module 14, Topic 3's specific "one philosopher reverses their order" fix, generalized into a systematic rule any real system can adopt.

Of these four, **attacking circular wait via a consistent global ordering** is by far the most commonly used in real-world practice — it's relatively simple to state as a rule ("always acquire locks in this specific, agreed-upon order") and doesn't require the more invasive changes (giving up exclusive access, or supporting rollback/preemption) the other three strategies need.

## Internal Working (Preview)

```
   Deadlock requires ALL FOUR simultaneously:

   1. Mutual exclusion   ┐
   2. Hold-and-wait        │  ALL FOUR present
   3. No preemption         │  → deadlock POSSIBLE
   4. Circular wait        ┘

   Deny ANY ONE → deadlock IMPOSSIBLE:

   Deny #4 (circular wait) via consistent global lock ordering:

     Rule: always acquire Lock A before Lock B, system-wide, no exceptions

     Thread 1: lock(A) → lock(B)   ✓ follows the rule
     Thread 2: lock(A) → lock(B)   ✓ follows the rule (NOT lock(B) then lock(A)!)

     No thread can ever be waiting on A while holding B, and simultaneously
     have another thread waiting on B while holding A — the specific
     structure required for a cycle simply cannot arise.
```

## Real-World Analogy

Think of the four conditions like four separate valves that must all be open simultaneously for water to flow through a specific dangerous pipe junction (deadlock). Closing any single valve stops the flow entirely, regardless of the other three. "Attacking circular wait" is like posting one single global rule at every junction in a large facility: "always route hot-water lines before cold-water lines, everywhere, with no exceptions" — a simple, uniform rule that, once followed consistently across the entire facility, makes the specific circular-blockage scenario structurally impossible, without needing to redesign the valves themselves (mutual exclusion), change how workers request access (hold-and-wait), or add a mechanism to forcibly override someone's in-progress work (preemption).

## Why Consistent Ordering Is the Preferred Real-World Strategy

Attacking mutual exclusion isn't always possible (some resources genuinely require exclusive access). Attacking hold-and-wait (acquire everything at once, or nothing) can be awkward when a thread doesn't know in advance exactly which resources it will need. Attacking no preemption requires the ability to safely roll back a thread's partial progress, which is often complex or outright impossible for real resources. Attacking circular wait via a simple, globally-agreed acquisition order, by contrast, requires only **discipline** (a rule every developer follows) rather than a fundamental change to how resources themselves behave — which is precisely why it's the dominant, most widely recommended real-world deadlock-prevention strategy.

## Advantages of Prevention Strategies

- **Deadlock becomes structurally impossible**, not merely unlikely — a genuine guarantee, not a probabilistic hope.
- **Consistent ordering, specifically, requires no runtime overhead or complex rollback machinery** — it's a discipline enforced at code-design time.

## Disadvantages / Trade-offs

- **Not always fully practical** — attacking hold-and-wait (acquire everything up front) can force a thread to acquire resources it might not end up needing on every code path, or requires knowing every needed resource in advance, which isn't always possible.
- **Consistent ordering requires organization-wide discipline** — a single piece of code anywhere in a large system that violates the agreed-upon lock order reintroduces the possibility of circular wait, and thus deadlock.

## Best Practices

- Establish and document a single, consistent global ordering for any locks that might ever be acquired together, and enforce it as a strict code-review requirement — this is the most broadly practical of the four prevention strategies for most real systems.
- When evaluating any code path that acquires more than one lock, explicitly check it against the established ordering rule — a violation anywhere is a real, potential deadlock, not just a style issue.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Deadlock can be prevented by fixing just one of the four conditions, but any one is as good as another to target." | While denying any single condition does prevent deadlock, the four strategies have very different practical costs — attacking circular wait via consistent ordering is generally far more practical than attacking mutual exclusion or no preemption, which often can't be changed at all for certain resources. |
| "As long as most of the code follows a consistent lock-acquisition order, deadlock is prevented." | A single inconsistent code path anywhere in the system is enough to reintroduce the possibility of circular wait — the ordering rule must be followed universally, with no exceptions, to guarantee prevention. |

## Interview Questions

1. **Q: What are the four necessary conditions for deadlock?**
   A: Mutual exclusion (a resource can only be held by one thread at a time), hold-and-wait (a thread holds one resource while waiting for another), no preemption (resources can't be forcibly taken away), and circular wait (a cycle of threads each waiting on a resource the next one holds).

2. **Q: Why does denying just one of the four conditions prevent deadlock entirely?**
   A: Because all four conditions must hold simultaneously for deadlock to be possible — if even one can never occur, the specific structure required for a deadlock cannot form, regardless of the state of the other three.

3. **Q: Why is attacking circular wait via a consistent global lock ordering the most commonly used real-world prevention strategy?**
   A: It requires only a disciplined, agreed-upon rule (acquire locks in a fixed order, everywhere) rather than a fundamental change to how resources behave — unlike attacking mutual exclusion (not always possible) or no preemption (requires complex, often infeasible rollback machinery).

## Summary

- Deadlock requires all four conditions simultaneously: mutual exclusion, hold-and-wait, no preemption, and circular wait.
- Module 14, Topic 3's dining philosophers deadlock satisfied all four; its fix specifically denied circular wait by breaking acquisition-order symmetry.
- Prevention strategies each deny one specific condition; attacking circular wait via a consistent, global lock-acquisition ordering is the most broadly practical approach in real-world systems.
- The next topic covers two further general strategies — avoidance and detection — for systems where strict prevention isn't practical or fully achievable.
