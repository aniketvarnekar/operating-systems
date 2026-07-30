# History of Operating Systems

## Learning Objectives

By the end of this section you should be able to:
- Describe the major eras of operating system design (batch processing, multiprogramming, time-sharing) in order
- Explain what specific hardware or economic pressure forced each era's transition to the next
- Connect each historical era to the modern OS concept it directly produced

## Prerequisites

- Topic 1 (What Is an Operating System?)

## Motivation

OS concepts can feel arbitrary — why do we have "processes"? Why "virtual memory"? These ideas were not invented in the abstract; each one was a direct, forced response to a real, expensive problem at the time. Knowing the history turns "arbitrary rules to memorize" into "obvious solutions to problems you can already see coming."

## Problem Statement

Early computers (1940s–1950s) had **no operating system at all**. A programmer would:
1. Physically sign up for a block of time on the one computer that existed.
2. Load their program (often via punch cards) directly into the machine.
3. Run it, alone, with full and exclusive control of the hardware.
4. Manually remove their materials so the next person could start their own turn.

This "one person, one program, one machine, in sequence" model wasted enormous amounts of the (extremely expensive, room-sized) computer's time — the machine sat idle every time a human was loading cards or reading printed output. As computers got faster and more expensive to build, idle time became an increasingly unacceptable waste of a scarce, costly resource. Every subsequent era of OS history is a response to squeezing more useful work out of expensive hardware.

## Concept

### Era 1: Batch Processing

Instead of one program running while a human fiddles with cards, an operator would collect a **batch** of many programs onto one tape or card deck ahead of time, and feed them to the computer one after another without a human in the loop between each one.

- **What it solved**: eliminated the idle time spent waiting on a human between individual runs.
- **What it didn't solve**: if one program in the batch entered an infinite loop or crashed, it could still take down or stall the entire batch — there was still no isolation between programs, and no way to interact with a running program.
- **Direct descendant today**: batch/cron jobs, CI pipelines — work queued up and run unattended, in sequence.

### Era 2: Multiprogramming

Batch processing solved the "loading" idle time, but exposed a new one: whenever a running program needed to wait on a slow I/O operation (like reading from a tape or an early disk), the CPU sat completely idle waiting for it — even though other programs in the batch were ready and waiting for CPU time.

**Multiprogramming** loaded several programs into memory *at once*, and when the currently running program blocked on slow I/O, the OS would switch the CPU to a different, ready program instead of leaving it idle.

- **What it solved**: the CPU is kept busy running *some* program's instructions almost continuously, instead of idling during another program's I/O wait.
- **What it required, for the first time**: real memory protection (so one program in memory couldn't corrupt another's data) and real CPU scheduling (a policy for choosing which ready program runs next) — the direct ancestors of Modules 04–11 in this course.

### Era 3: Time-Sharing

Multiprogramming made better use of the CPU, but a program still had to wait for its entire batch-scheduled turn — there was no notion of interacting with a running program moment-to-moment, the way you use a computer today.

**Time-sharing** systems (pioneered in the 1960s) let the OS rapidly switch between many programs — often dozens of users' programs — giving each one a very short slice of CPU time in turn, fast enough that from each individual user's perspective, it *felt* as though they had the whole machine to themselves, responding instantly to their input.

- **What it solved**: real, interactive, "feels immediate" use of a computer by multiple simultaneous users — the direct ancestor of every modern desktop, laptop, and phone, all of which run dozens to thousands of processes "at once" from your perspective.
- **What it required**: fast, cheap context switching (Module 03) and scheduling policies that specifically prioritize responsiveness (Modules 04–05), not just raw throughput.

### The Modern Era

Modern general-purpose operating systems (Linux, Windows, macOS) are, at their core, direct engineering descendants of time-sharing systems — refined over decades with better memory virtualization (Modules 06–11), richer concurrency primitives (Modules 12–16), and robust crash-resistant persistence (Modules 17–23), but solving fundamentally the same three problems: virtualization, concurrency, and persistence (Topics 3–5 of this module).

## Internal Working (Preview)

```
 Batch Processing         Multiprogramming          Time-Sharing
 ─────────────────        ─────────────────         ─────────────────
 Program A runs           Program A runs,            Program A, B, C, ...
 to completion,           blocks on I/O →             each get a short
 then Program B            OS switches CPU             CPU time-slice,
 loads and runs.           to Program B.               rapidly rotating.

 CPU idle during           CPU idle only if           CPU (almost) never
 card loading.             *no* program is             idle; interactive
                           ready to run.               feel for every user.
```

## Real-World Analogy

Think of a single hair salon with one chair (the CPU):

- **Batch processing**: clients are seen strictly one at a time, start to finish, in a pre-arranged order — but if a client steps out mid-appointment (analogous to slow I/O), the chair sits empty until they return.
- **Multiprogramming**: the stylist keeps a second client warming up; the moment the first client steps out, the stylist starts on the second client instead of waiting idle.
- **Time-sharing**: the stylist rapidly rotates between several clients in short bursts — a few minutes on client A's hair, switch to client B, switch to client C, back to A — fast enough that every client feels like they're being continuously attended to, even though it's really one stylist serving all of them in tiny interleaved slices.

## Why OS Design Evolved This Way

Each transition was forced by the same underlying economic pressure: **computer hardware was extremely expensive relative to human time**, so every era's improvement is really "waste less of the computer's time, even if it costs more engineering effort or slightly worse latency for an individual task." That pressure — squeeze more useful work out of scarce, shared hardware — is the same pressure that produces virtualization, scheduling, and concurrency control in every module that follows.

## Advantages of Understanding This History

- Explains *why* processes, scheduling, and memory protection exist at all — they were forced by real idle-time and safety problems, not designed in a vacuum.
- Makes trade-offs in later modules (e.g., throughput vs. responsiveness in scheduling, Module 04) intuitive, because you already know they map to "batch-style efficiency" vs. "time-sharing-style interactivity" goals that have been in tension since the 1960s.

## Best Practices

- When a modern OS design choice seems overly conservative or defensive (e.g., strict memory isolation between processes), remember it usually exists because an earlier, less careful era learned that lesson the hard way.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Multiprogramming and multitasking/time-sharing are the same thing." | Multiprogramming just keeps the CPU busy across several loaded programs; time-sharing specifically optimizes for fast, interactive switching so it *feels* instantaneous to a human user. Time-sharing is a stricter, later refinement built on multiprogramming's foundation. |
| "Old batch systems didn't need an OS." | They needed a simpler one — but even automatically feeding one program after another from a batch, without a human in the loop, is itself a (primitive) operating system responsibility. |

## Interview Questions

1. **Q: What specific problem did multiprogramming solve that batch processing didn't?**
   A: Batch processing still let the CPU sit idle whenever the currently running program was blocked on slow I/O. Multiprogramming kept multiple programs in memory simultaneously so the OS could switch the CPU to a different, ready program during that idle wait.

2. **Q: What's the key difference between multiprogramming and time-sharing?**
   A: Multiprogramming's goal is keeping the CPU busy (throughput). Time-sharing's goal is making the machine feel simultaneously, instantly responsive to multiple interactive users, via fast, frequent switching between short time-slices — a stricter goal that multiprogramming alone doesn't guarantee.

3. **Q: Why did real memory protection become necessary specifically starting in the multiprogramming era?**
   A: Once more than one program was resident in memory at the same time, an unprotected program could read or corrupt another program's memory — a risk that simply didn't exist when only one program at a time was ever loaded, as in pure batch processing.

## Summary

- Batch processing eliminated idle time between manual program loads, but left the CPU idle during I/O and offered no isolation.
- Multiprogramming kept the CPU busy across I/O waits by holding multiple programs in memory, forcing the invention of real memory protection and scheduling.
- Time-sharing further optimized for fast, interactive responsiveness across many simultaneous users, forcing cheap context switching and responsiveness-aware scheduling.
- Every modern OS is a direct descendant of time-sharing design, still solving the same underlying problems: virtualization, concurrency, and persistence (Topics 3–5).
