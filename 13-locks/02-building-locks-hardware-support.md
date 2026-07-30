# Building Locks: Hardware Support

## Learning Objectives

By the end of this section you should be able to:
- Explain why a lock built from ordinary load/store instructions alone is incorrect
- Explain what the test-and-set instruction does, and why its atomicity is the key property
- Explain how compare-and-swap generalizes test-and-set

## Prerequisites

- Topic 1 (The Basic Idea of Locks)

## Motivation

Topic 1 introduced lock() and unlock() as if they were simple, given primitives. But lock() has to somehow check "is this lock currently free?" and, if so, mark it as held — and that check-then-set sequence is itself a critical section, facing exactly the same interleaving danger locks were invented to solve in the first place. This topic shows why this is a real, non-trivial problem, and the specific hardware support that resolves it.

## Problem Statement

Suppose you tried to implement a lock using only an ordinary shared variable and ordinary load/store instructions:

```c
// naive, INCORRECT attempt
while (flag == 1) { }  // "spin" while someone else holds the lock
flag = 1;               // now I hold it
```

This looks reasonable, but consider two threads both reaching the `while` check at nearly the same moment, when `flag` is currently 0. Both threads' `while (flag == 1)` check passes (flag is 0 for both, momentarily), and **both** then proceed to set `flag = 1` and enter the critical section simultaneously — the exact race condition locks were supposed to prevent, just moved one level down, into the lock's own implementation. The "check, then act" sequence (checking if the flag is free, then setting it) is not itself atomic using ordinary instructions.

## Concept

### Why Ordinary Instructions Cannot Build a Correct Lock

The fundamental issue: checking a condition and then acting on it (updating shared state based on that check) requires **at least two separate steps** — and if those two steps can be interrupted (by a context switch, Module 03, Topic 4, or by another thread running on a different core simultaneously) between the check and the act, another thread can slip in and violate the intended guarantee. This is precisely the same class of problem as Module 01, Topic 4's lost update — except now it's happening *inside* the very mechanism meant to prevent that problem, which is why ordinary load/store instructions alone are insufficient no matter how cleverly arranged.

### Test-and-Set: Atomic Check-and-Set in One Hardware Instruction

> The **test-and-set** instruction is a single CPU instruction that atomically reads a memory location's *old* value, writes a *new* value into it, and returns the old value — all as one indivisible hardware operation that cannot be interrupted or interleaved with any other thread's instructions, even on a different CPU core.

```
 old_value = test_and_set(&flag, 1)
 // atomically: old_value = flag; flag = 1;   (as ONE indivisible step)
```

A lock built on test-and-set works like this:

```c
void lock(int *flag) {
    while (test_and_set(flag, 1) == 1) {
        // someone else already held it (old value was 1) — keep trying
    }
    // old value was 0 — we successfully claimed the lock
}

void unlock(int *flag) {
    *flag = 0;
}
```

Because test-and-set's read-and-write happens as one indivisible hardware step, there is no window where two threads can both see the lock as free and both proceed — whichever thread's test-and-set instruction executes first will see the old value 0 (success) and set it to 1; any other thread's test-and-set, no matter how closely timed, will see the value already set to 1 (failure) and keep retrying (this repeated retrying is called **spinning**, covered further in Topic 3).

### Compare-and-Swap: A More General Primitive

> **Compare-and-swap (CAS)** atomically checks whether a memory location currently holds an *expected* value, and if so, updates it to a *new* value — reporting whether the update succeeded — again, as one indivisible hardware step.

```
 success = compare_and_swap(&flag, expected=0, new=1)
 // atomically: if (flag == expected) { flag = new; return true; }
 //             else                   { return false; }
```

CAS is strictly more general than test-and-set: it can implement the same simple lock (checking for an expected "free" value of 0 before setting it to 1), but it can also implement more sophisticated lock-free data structures and algorithms (briefly previewed here, more relevant to advanced concurrent data structure design) where a thread needs to update a value only if it hasn't been changed by someone else since it was last read — a pattern that test-and-set alone cannot express as directly.

## Internal Working (Preview)

```
   NAIVE (incorrect) check-then-set — NOT atomic:

     Thread A: read flag (0) ──────────► write flag = 1
     Thread B:      read flag (0) ──────────► write flag = 1
                          ▲
                  BOTH threads saw flag=0 — BOTH proceed. BROKEN.


   test_and_set — ATOMIC, hardware-enforced:

     Thread A: test_and_set(flag, 1) → reads 0, sets 1, returns 0 (SUCCESS)
                    [ this happens as ONE indivisible hardware step ]
     Thread B: test_and_set(flag, 1) → reads 1 (A already set it),
                                         sets 1 again, returns 1 (FAILURE,
                                         keeps spinning)
```

## Real-World Analogy

Think of the naive, broken approach like two people separately peeking through a door's keyhole to see if a room is empty, and only *then* walking in — if both peek at almost the same instant, both see "empty" and both walk in simultaneously, since peeking and walking in are two separate actions with a gap between them. Test-and-set is like a smart lock mechanism that combines "check if the room is empty" and "claim it as occupied" into one single, physically indivisible motion — like a turnstile that only lets one person's claim register at a time, no matter how close together two people arrive; whoever's claim registers first gets in, and the turnstile itself guarantees there's no gap where both claims could succeed.

## Why Hardware Support Is Genuinely Necessary

This isn't a matter of writing sufficiently clever software — the fundamental problem is that any check-then-act sequence built from separate instructions has a real, physical gap between the "check" and the "act," a gap during which a context switch (Module 03, Topic 4) or a different CPU core's simultaneous instruction can interleave. Only a hardware guarantee — that a specific instruction's read-and-write happens as one truly indivisible unit, uninterruptible even across CPU cores — can close this gap entirely. This is directly analogous to why the MMU's translation (Module 07, Topic 1) had to be hardware rather than software: some guarantees are only achievable with dedicated hardware support, not by rearranging ordinary instructions.

## Advantages of Hardware-Based Atomic Instructions

- **Genuinely, provably correct mutual exclusion** — unlike the naive check-then-set approach, there is no gap during which two threads can both succeed.
- **A small, general building block** — test-and-set and compare-and-swap are primitive enough to be implemented efficiently in hardware, yet powerful enough to build correct locks (and more advanced concurrent structures) on top of.

## Disadvantages / Costs

- **Repeatedly retrying (spinning) while waiting has a real cost** — a thread failing test-and-set/CAS in a loop consumes CPU cycles doing nothing useful, a cost examined directly in Topic 3.
- **Contention on the same atomic variable across many CPU cores can itself become a bottleneck** — many cores repeatedly attempting test-and-set/CAS on the same memory location can create real hardware-level contention, a concern in highly parallel systems.

## Best Practices

- When explaining why locks can't simply be built from ordinary variables, lead directly with the check-then-act gap — it's the same fundamental issue as Module 01, Topic 4's lost update, just one level deeper in the stack.
- Recognize test-and-set and compare-and-swap as the foundational building blocks beneath essentially every higher-level lock or concurrent data structure — even lock implementations you never see the internals of are almost certainly built on one of these two primitives.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A lock can be correctly implemented using just an ordinary shared boolean variable and normal if-checks." | Checking the variable and then setting it are two separate steps with a real gap between them; two threads can both pass the check before either has set the variable, both proceeding into the critical section — exactly the race condition a lock is supposed to prevent. |
| "Test-and-set and compare-and-swap are just software conventions, not different from a regular function call." | They are specific hardware instructions guaranteeing atomicity — an indivisible read-and-write — a guarantee that cannot be achieved by any sequence of ordinary, separately-executed instructions, no matter how they're arranged. |

## Interview Questions

1. **Q: Why can't a correct lock be built using only ordinary load and store instructions?**
   A: Checking whether the lock is free and then marking it as held are two separate steps; if a context switch or a concurrent instruction on another core occurs between them, two threads can both see the lock as free and both proceed, resulting in the exact race condition the lock was meant to prevent.

2. **Q: What does the test-and-set instruction do, and why does its atomicity matter?**
   A: It atomically reads a memory location's old value and writes a new value into it as one indivisible hardware step, returning the old value. Its atomicity closes the gap that makes naive check-then-set approaches incorrect — no other thread's instruction can be interleaved between the read and the write.

3. **Q: How does compare-and-swap generalize test-and-set?**
   A: Compare-and-swap atomically updates a memory location to a new value only if it currently holds an expected value, reporting success or failure — it can implement the same simple lock as test-and-set, but also supports more general patterns, like updating a value only if it hasn't changed since it was last read.

## Summary

- A lock cannot be correctly built from ordinary load/store instructions alone, because checking a condition and acting on it are separate steps with a real interleaving gap between them.
- Test-and-set is a hardware instruction that atomically reads and writes a memory location as one indivisible step, closing this gap and enabling a correct spinning lock.
- Compare-and-swap generalizes this further, atomically updating a value only if it matches an expected value — a more flexible primitive underlying many concurrent algorithms.
- The next topic addresses what a thread should actually do while waiting for a lock it can't yet acquire — spinning has a real cost, and real systems use a hybrid approach to manage it.
