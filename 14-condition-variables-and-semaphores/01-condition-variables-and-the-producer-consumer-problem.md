# Condition Variables and the Producer-Consumer Problem

## Learning Objectives

By the end of this section you should be able to:
- Explain why a lock alone cannot express "wait until a condition becomes true"
- Describe the producer-consumer (bounded-buffer) problem precisely
- Explain what wait() and signal() do on a condition variable, and why wait() must be called while holding the associated lock

## Prerequisites

- Module 13 in full

## Motivation

Module 13's locks solve exactly one problem: ensuring only one thread is inside a critical section at a time. But many real coordination problems need something more: a thread that wants to proceed only once some *specific condition* is true — not just "no one else is using this," but "there's actually an item ready for me to consume." This topic introduces the tool built specifically for that.

## Problem Statement

Consider a shared, fixed-size buffer: one or more **producer** threads add items to it, and one or more **consumer** threads remove items from it. Two problems need solving simultaneously:

1. A consumer must not try to remove an item from an **empty** buffer.
2. A producer must not try to add an item to a **full** buffer.

A lock (Module 13) can protect the buffer's internal state from simultaneous, uncoordinated modification — but it cannot express "wait until the buffer actually has something in it" or "wait until the buffer actually has room." A thread could acquire the lock, find the buffer empty, and... then what? Releasing the lock and immediately reacquiring it in a tight loop (checking over and over) would technically work, but wastes CPU exactly like a spin lock waiting on a long hold (Module 13, Topic 3) — and worse, it doesn't generalize cleanly to being efficiently *woken up* right when the condition actually changes.

## Concept

### The Producer-Consumer (Bounded-Buffer) Problem

> The **producer-consumer problem** (or bounded-buffer problem) asks how to coordinate producer threads adding items to a shared, fixed-capacity buffer and consumer threads removing items from it, such that producers never add to a full buffer, consumers never remove from an empty buffer, and access to the buffer's internal state is always properly synchronized.

This is one of the most common, recurring patterns in real concurrent systems — a background thread producing log entries for another thread to write to disk, a network thread receiving data for a processing thread to consume, and countless similar structures.

### Condition Variables

> A **condition variable** is a synchronization primitive that lets a thread **sleep** (voluntarily give up the CPU, moving to Blocked, Module 02, Topic 2) while waiting for some condition to become true, and lets another thread **wake** one (or all) sleeping threads once that condition changes — without the waiting thread needing to repeatedly, wastefully re-check the condition itself (avoiding the wasteful spin-loop from the Problem Statement).

A condition variable provides two core operations:

- **wait(condition_variable, lock)**: atomically releases the given lock and puts the calling thread to sleep, waiting on this condition variable. When later woken, it re-acquires the lock before returning.
- **signal(condition_variable)** (sometimes called `notify`): wakes up (at least) one thread currently sleeping on this condition variable, if any.

### Why wait() Must Release the Lock Atomically

This is the single most important, and most easily gotten-wrong, detail: a thread calling `wait()` must be holding the associated lock beforehand (since it's about to check some shared, lock-protected condition), but it must **release that lock as part of going to sleep** — otherwise, no other thread could ever acquire the lock to change the condition and call `signal()`, and the sleeping thread would wait forever. This release-and-sleep must happen as one atomic operation: if there were any gap between releasing the lock and actually going to sleep, another thread could change the condition and call `signal()` in that exact gap, and the waiting thread — not yet actually asleep on the condition variable — could miss that wakeup entirely and sleep forever despite the condition already being true. A correct condition variable implementation guarantees this release-and-sleep happens as one indivisible step.

### The Standard Pattern: Always Re-Check in a While Loop

A woken thread must **re-check the condition in a loop**, not simply assume it's now true:

```c
lock(&mutex);
while (buffer_is_empty()) {
    wait(&cond, &mutex);   // atomically: release mutex, sleep; re-acquire mutex on wake
}
// buffer is confirmed non-empty here — safe to consume
remove_item();
unlock(&mutex);
```

This `while` (not `if`) is deliberate: by the time a woken thread re-acquires the lock, another thread might have already changed the condition again (e.g., a different consumer thread, woken at the same time, might have already consumed the one available item first) — re-checking the actual condition after waking, rather than trusting that being woken means the condition is still true, is the only safe pattern.

## Internal Working (Preview)

```
   Consumer thread:                    Producer thread:

   lock(&mutex)                        lock(&mutex)
   while (buffer empty)                add item to buffer
       wait(&cond, &mutex)  ◄────┐     signal(&cond)  ───────┐
       [ ATOMIC: release          │                            │
         mutex + sleep ]          │                            │
                                    │                            ▼
       [ woken up here ]           │                  wakes the sleeping
       [ re-acquire mutex ]        │                  consumer thread
   (loop re-checks condition) ─────┘
   remove item
   unlock(&mutex)
```

## Real-World Analogy

Think of a customer waiting at a restaurant's pickup counter for an order that isn't ready yet. Instead of standing there constantly asking "is it ready? is it ready?" (wasteful spinning), the customer sits down and waits to be called by name (sleeping on a condition variable) — the counter staff, once the order genuinely is ready, calls that customer's name (signal()), waking them specifically to come collect it. Crucially, when the customer sits down to wait, they aren't physically blocking the counter (they release the lock/mutex) — otherwise, staff could never get to the counter to prepare or announce anything in the first place. And a sensible customer, upon hearing *any* name called, double-checks it's actually theirs before walking up (re-checking the condition in a while loop) — since multiple people might be waiting, and being called doesn't automatically guarantee it's specifically your order that's ready.

## Why Condition Variables Are Necessary

A lock alone (Module 13) only ever answers "can I access this shared data right now, exclusively?" — it has no concept of "wait until some specific fact about that data becomes true." Without condition variables, achieving the same effect would require either wasteful busy-waiting (repeatedly locking, checking, unlocking in a tight loop — the exact wasteful spinning Module 13, Topic 3 examined) or an ad hoc, error-prone, hand-rolled signaling mechanism. Condition variables provide a standard, correctly-synchronized way to sleep until notified, integrated directly with the lock that protects the condition being waited on.

## Advantages of Condition Variables

- **No wasted CPU cycles waiting** — a thread genuinely sleeps (Blocked, Module 02, Topic 2) rather than busy-checking, freeing the CPU for other useful work while it waits.
- **Directly solves the producer-consumer problem** — and, more generally, any "wait until X becomes true" coordination need, cleanly integrated with the lock protecting the shared state.

## Disadvantages / Risks

- **Easy to get subtly wrong** — forgetting to hold the lock before calling wait(), or checking the condition with `if` instead of `while`, are both real, common, and dangerous mistakes that can cause missed wakeups or acting on a stale condition.
- **signal() only guarantees waking *a* waiting thread (or none, if there are none waiting), not necessarily immediately, or in any particular order among multiple waiters** — code must not assume anything stronger than "the condition should be re-checked."

## Best Practices

- Always call wait() inside a `while` loop that re-checks the actual condition, never inside an `if` — a woken thread must never assume the condition is still true without re-verifying it itself.
- Always hold the associated lock both before calling wait() and while calling signal() — this is required for the atomicity guarantees condition variables depend on to hold correctly.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "A thread can just check a shared condition without holding a lock, then call wait() if it's false." | Checking the condition and calling wait() must happen while holding the lock, so that no other thread can change the condition and call signal() in the gap between the check and going to sleep — otherwise the wakeup could be missed entirely. |
| "Using `if (buffer_is_empty())` instead of `while (buffer_is_empty())` before wait() is fine." | By the time a woken thread re-acquires the lock, the condition may have changed again (e.g., another thread consumed the item first) — an `if` doesn't re-verify this, risking acting on a stale, no-longer-true condition; `while` is required for correctness. |

## Interview Questions

1. **Q: What problem do condition variables solve that a plain lock cannot?**
   A: A lock only guarantees exclusive access to shared data; it cannot express "wait until some specific condition about that data becomes true." Condition variables let a thread sleep until notified that the condition may have changed, avoiding both a missing coordination mechanism and wasteful busy-waiting.

2. **Q: Why must wait() atomically release the lock as part of going to sleep?**
   A: If there were any gap between releasing the lock and actually sleeping, another thread could change the condition and call signal() in that exact gap, and the waiting thread — not yet actually registered as sleeping — could miss the wakeup and sleep forever despite the condition already being true.

3. **Q: Why should a woken thread re-check its condition in a while loop rather than assuming it's true after being woken?**
   A: Because by the time it re-acquires the lock, another thread may have already changed the condition again (e.g., a different waiting thread already consumed the available item) — re-checking is the only way to safely confirm the condition still holds before proceeding.

## Summary

- The producer-consumer (bounded-buffer) problem requires producers and consumers to coordinate around a shared buffer's fullness/emptiness, not just its exclusive access.
- A condition variable lets a thread sleep (via wait(), atomically releasing its lock) until another thread signals that the condition may have changed, avoiding wasteful busy-waiting.
- A woken thread must always re-check its condition in a while loop, since it may no longer hold by the time the thread reacquires the lock.
- The next topic introduces semaphores, a single, more general primitive that can express both mutual exclusion (Module 13) and condition-variable-style waiting (this topic) using one consistent interface.
