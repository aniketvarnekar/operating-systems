# Deadlock Avoidance and Detection

## Learning Objectives

By the end of this section you should be able to:
- Explain the difference between deadlock prevention, avoidance, and detection as three distinct strategic approaches
- Explain, at a high level, what a "safe state" is and how avoidance uses this concept
- Explain how detection-and-recovery differs fundamentally from both prevention and avoidance

## Prerequisites

- Topic 2 (Deadlock: Conditions and Prevention)

## Motivation

Topic 2 covered prevention — structurally guaranteeing deadlock can never occur by denying one of its four necessary conditions. This topic covers two further, genuinely different strategic postures a system can take instead: proactively steering around danger without denying any condition outright (avoidance), or accepting that deadlock might happen and dealing with it after the fact (detection and recovery).

## Problem Statement

Prevention (Topic 2) works by structurally ruling out one of the four necessary conditions, system-wide, at design time. But what if a system can't easily commit to a strict, global rule like "always acquire locks in this fixed order" — perhaps because which resources a thread will need isn't known until runtime? Are there other legitimate strategies for handling deadlock risk?

## Concept

### Deadlock Avoidance: Steering Around Unsafe States at Runtime

> **Deadlock avoidance** does not deny any of the four necessary conditions outright. Instead, it requires threads to declare their resource needs in advance, and the system uses this information at runtime to decide, before granting any specific resource request, whether granting it could ever lead to a state from which deadlock becomes unavoidable — refusing (delaying) the request if so.

The core concept: a **safe state** is one from which the system can guarantee every thread can eventually complete, no matter what order they end up requesting their remaining declared resources in. Before granting a resource request, an avoidance algorithm (such as the classic **Banker's Algorithm**) checks whether doing so would leave the system in a safe state — one where at least one valid completion ordering for all threads still exists — and only grants the request if it would.

This is meaningfully different from prevention: prevention makes deadlock structurally impossible by ruling out a condition entirely, regardless of runtime behavior. Avoidance allows all four conditions to potentially exist, but uses runtime information (each thread's declared maximum resource needs) to carefully steer around ever entering an actually dangerous configuration.

### The Practical Limitation of Avoidance

Avoidance requires each thread to know and declare its **maximum possible resource need in advance** — information that's often genuinely unavailable in real, general-purpose systems (a program frequently doesn't know exactly which locks it will need to acquire until it's already running and reacting to actual conditions). This is precisely why avoidance, while theoretically elegant, is used far less often in general-purpose operating systems than the simpler, more broadly applicable prevention strategies from Topic 2 — it's more commonly seen in specialized, constrained settings (like certain embedded or real-time systems) where resource needs genuinely can be known ahead of time.

### Deadlock Detection and Recovery: Let It Happen, Then Fix It

> **Deadlock detection** takes an entirely different posture: it does not try to prevent or avoid deadlock at all. Instead, it lets the system run normally, periodically checks (using an algorithm that looks for a cycle in a graph of which threads are waiting on which resources, held by which other threads) whether a deadlock currently exists, and if one is found, takes **recovery** action — most commonly, forcibly terminating (or rolling back the progress of) one or more of the deadlocked threads, releasing their held resources so the remaining threads can proceed.

This strategy accepts deadlock as a real, occasional possibility rather than trying to structurally rule it out, trading the overhead and restrictiveness of prevention/avoidance for periodic detection overhead and the (sometimes acceptable, sometimes costly) consequence of having to terminate or roll back a deadlocked thread's work when detection does find one.

### When Each Strategy Makes Sense

- **Prevention** (Topic 2) is the most broadly practical default for most general-purpose software — a simple, disciplined rule (consistent lock ordering) with no runtime overhead.
- **Avoidance** fits specialized settings where resource needs genuinely can be declared in advance, and the overhead of runtime safe-state checking is acceptable.
- **Detection and recovery** fits settings where deadlock is expected to be rare, prevention/avoidance would be overly restrictive or impractical to enforce, and the cost of occasionally terminating/rolling back a thread is acceptable relative to the flexibility gained elsewhere.

## Internal Working (Preview)

```
   PREVENTION (Topic 2): deny a condition — deadlock structurally IMPOSSIBLE
        (e.g., consistent lock ordering, enforced at design time)

   AVOIDANCE: allow all 4 conditions to exist, but check EVERY resource
              request at runtime against declared future needs —
              refuse any request that would enter an UNSAFE state

        Request comes in → "would granting this still leave at least
                             one valid completion order for everyone?"
                                  │
                            ┌─────┴─────┐
                           YES           NO
                            │             │
                          GRANT        DELAY/REFUSE

   DETECTION: let the system run freely — PERIODICALLY scan for a
              cycle in the "who's waiting on whom" graph

        Cycle found? → RECOVER: kill or roll back one of the
                        deadlocked threads, freeing its resources
```

## Real-World Analogy

**Prevention** is like a building code that simply forbids a specific, dangerous type of wiring configuration outright — it can never be built that way, full stop. **Avoidance** is like an electrician who, before connecting any new device, first checks the building's full declared future power needs and only proceeds if doing so still leaves a guaranteed-safe configuration for everything else that might eventually be plugged in — more careful and flexible than an outright ban, but requiring everyone to honestly declare their future needs up front. **Detection and recovery** is like a building that runs normally without any of these upfront checks, but has a smoke detector that periodically checks for an actual fire — and if one is found, the fire department is called in to put it out (recovery), accepting the occasional real cost of a fire in exchange for not restricting how the building is used day to day.

## Why All Three Strategies Coexist as Legitimate Options

There is no single universally correct choice among prevention, avoidance, and detection — each makes a different trade-off between upfront restrictiveness, runtime overhead, required advance knowledge, and the acceptable cost of occasionally needing recovery. A general-purpose OS kernel might favor prevention-by-convention for its own internal locking (Topic 2), a specialized real-time system with fully-known resource needs might favor avoidance, and a large distributed database might favor detection-and-recovery (periodically checking for deadlocked transactions and aborting/rolling back one to let others proceed) specifically because prevention's strict global lock ordering is impractical to enforce across many independently-developed application components.

## Advantages and Disadvantages by Strategy

- **Prevention**: no runtime overhead, deadlock genuinely impossible — but can be restrictive or require sweeping, disciplined changes to how resources are acquired system-wide.
- **Avoidance**: more flexible than an outright ban, allows all four conditions to coexist safely — but requires advance knowledge of maximum resource needs, which is often unavailable, and has real runtime checking overhead.
- **Detection and recovery**: no upfront restriction on how threads acquire resources at all — but requires accepting the real cost of periodically checking for cycles and occasionally terminating or rolling back a thread's progress when deadlock is actually found.

## Best Practices

- Default to prevention (Topic 2's consistent lock ordering) for general-purpose concurrent code, unless a specific, well-understood reason favors avoidance or detection instead.
- Reserve avoidance for settings where resource needs genuinely can be declared reliably in advance; reserve detection-and-recovery for settings where occasional termination/rollback is an acceptable cost and where prevention's discipline can't realistically be enforced system-wide (e.g., across independently-developed components in a large distributed system).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Deadlock avoidance and deadlock prevention are the same strategy with different names." | Prevention structurally denies one of the four necessary conditions, making deadlock impossible regardless of runtime behavior. Avoidance allows all four conditions to exist but uses runtime checks against declared future needs to steer around ever entering an actually unsafe state — a fundamentally different mechanism. |
| "Deadlock detection means the system prevents deadlock from ever completing." | Detection explicitly allows deadlock to actually occur; it only periodically checks for it after the fact and then recovers (typically by terminating or rolling back a deadlocked thread) — it does not prevent the deadlock state from forming in the first place. |

## Interview Questions

1. **Q: What's the fundamental difference between deadlock prevention and deadlock avoidance?**
   A: Prevention structurally denies one of the four necessary conditions for deadlock, making it impossible regardless of runtime behavior. Avoidance allows all four conditions to potentially exist, but uses runtime information (each thread's declared maximum future resource needs) to refuse any request that would leave the system in an unsafe state, from which deadlock could become unavoidable.

2. **Q: What is a "safe state" in the context of deadlock avoidance?**
   A: A state from which the system can guarantee every thread can eventually complete, regardless of the order in which they request their remaining declared resource needs — an avoidance algorithm only grants a new request if doing so still leaves the system in such a state.

3. **Q: How does deadlock detection and recovery differ from both prevention and avoidance?**
   A: It doesn't try to prevent or avoid deadlock at all — it lets the system run freely, periodically checks for an actual cycle of mutual waiting, and if one is found, recovers by terminating or rolling back one or more of the deadlocked threads to free their resources for the rest.

## Summary

- Deadlock avoidance allows all four necessary conditions to exist, but uses runtime checks against declared future resource needs to avoid ever entering an unsafe state — more flexible than prevention, but requiring advance knowledge that's often unavailable in general-purpose systems.
- Deadlock detection and recovery lets deadlock actually occur, periodically checks for it via cycle detection, and recovers by terminating or rolling back a deadlocked thread when found.
- Prevention, avoidance, and detection are three legitimate, genuinely different strategies, each fitting different practical circumstances — there is no single universally correct choice.
- This closes out the module's coverage of concurrency bugs — the module summary ties non-deadlock bugs and all of deadlock's handling strategies together, completing Concurrency's core toolkit before Module 16 covers an alternative concurrency model entirely: event-based concurrency.
