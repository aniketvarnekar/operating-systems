# MLFQ Refinements and Gaming Resistance

## Learning Objectives

By the end of this section you should be able to:
- Explain the starvation problem in basic MLFQ and how periodic priority boosting fixes it
- Explain how a process could "game" the basic rules, and how accounting for total time-at-level (rather than resetting on every yield) closes that loophole
- Explain why real-world scheduler design often looks like "a clean core idea, plus several targeted patches for adversarial edge cases"

## Prerequisites

- Topic 1 (The Multi-Level Feedback Queue)

## Motivation

Topic 1's basic MLFQ rules are elegant, but elegant rules and robust rules are not automatically the same thing. This topic walks through two concrete, realistic scenarios that break the basic rules — one accidental (starvation under heavy interactive load), one deliberate (a process gaming the system) — and shows the specific, minimal refinements that fix each one.

## Problem Statement

**Scenario 1 — Starvation:** Imagine a system with a constant, never-ending stream of new interactive jobs — each one starts at the highest priority level (Topic 1, Rule 3), and if there's always at least one such job ready to run, the scheduler (which always checks the highest non-empty queue first) might *never* get around to running anything sitting at a lower priority level. A long-running job that was fairly demoted for legitimately using full time slices could, in the worst case, wait an unbounded amount of time — real starvation, not just a slow turnaround.

**Scenario 2 — Gaming:** Recall Topic 1's Rule 5: a job that voluntarily yields before its time slice ends keeps its current priority. Imagine a deliberately (or accidentally, via a bug) crafted process that runs for 99% of its time slice, then issues a trivially fast, essentially meaningless I/O operation (yielding the CPU for a near-zero amount of time) just before the slice would expire. Under the basic rules as stated, this process is never actually demoted — it keeps "voluntarily yielding" just in time, every single round — while still consuming nearly an entire time slice's worth of CPU on every turn. This is functionally almost identical to a long-running CPU-bound job monopolizing high-priority treatment it was never meant to receive.

## Concept

### Fix for Starvation: Periodic Priority Boost

> **Priority boosting** periodically moves **every** process in the system back up to the highest priority queue, regardless of its recent history, at a fixed time interval.

This directly guarantees that no process can be starved indefinitely: even a job stuck at the lowest priority level is guaranteed to be bumped back to the top within, at most, one boost interval — after which it competes on equal footing again, and the whole demotion process (Topic 1's Rules 4–5) starts over from a clean slate. This also has a secondary benefit: a long-running job whose *behavior* has genuinely changed (say, it shifts from purely computational work to suddenly making frequent I/O calls) gets a fair chance to be re-classified correctly, rather than being permanently stuck with whatever priority its earlier behavior happened to earn it.

The length of the boost interval creates a trade-off directly analogous to Round Robin's time-slice trade-off (Module 04, Topic 5): too frequent, and long-running jobs never get properly demoted long enough to converge toward fair, lower-priority treatment; too infrequent, and a starved job could still wait an uncomfortably long time between boosts.

### Fix for Gaming: Track Total Time Used at a Level, Not Just "Did It Yield This Round"

> Instead of resetting a job's accounting the instant it voluntarily yields, a more robust MLFQ tracks the **total amount of CPU time a job has consumed at its current priority level, across all its turns combined** — regardless of how many times it yielded in between. Once that *cumulative* total reaches the equivalent of one full time slice's worth of usage, the job is demoted — even if it never used one single, uninterrupted turn all the way to the slice's exact end.

This closes the loophole from Scenario 2 directly: a process that repeatedly uses 99% of a time slice, yields briefly, and returns, still accumulates real CPU usage over time — under this refined accounting, it gets demoted once that accumulated usage crosses the threshold, exactly as if it had used one long, uninterrupted slice, regardless of how it artificially fragmented its own usage to game the naive per-round rule.

## Internal Working (Preview)

```
 Starvation fix — periodic priority boost:

   every BOOST_INTERVAL:
       move ALL jobs back to the highest priority queue
       (their prior demotion history is wiped clean)

 Gaming fix — cumulative time accounting per level:

   naive (gameable):     "did you yield before your slice ended THIS round?"
   robust (not gameable): "how much TOTAL CPU time have you used at this
                            priority level, summed across every round,
                            since you last changed levels?"
                           → demote once this total crosses one slice's worth,
                             regardless of how many small yields fragmented it
```

## Real-World Analogy

**Priority boost** is like a company that periodically resets everyone's "seniority-based perks" at a fixed interval (say, a fresh start every fiscal quarter) — even an employee who's been stuck with lower-priority treatment for a while gets a clean slate at the start of the new quarter, guaranteeing no one is permanently locked out of good treatment forever, and giving people whose actual role/behavior changed a fair chance to be reassessed.

**Cumulative accounting against gaming** is like a gym membership that tracks your *total* minutes of equipment usage across an entire day, rather than resetting the moment you briefly step away — someone trying to "hold" a popular machine all day by stepping away for one second every ten minutes doesn't actually escape the gym's fair-usage policy, because the policy sums up their total real usage time rather than being fooled by the brief, technically-compliant gaps.

## Why These Refinements Are Necessary

The basic MLFQ rules (Topic 1) are correct in spirit but naively trust that a process's yielding behavior always reflects genuine short-burst, interactive work — an assumption that both ordinary heavy interactive load (Scenario 1) and deliberate or accidental gaming (Scenario 2) can violate. Real OS schedulers must be designed defensively: not just "what's the elegant common case," but "what happens under sustained load, and what happens if a process's behavior is specifically engineered (intentionally or not) to exploit the letter of the rules while violating their intent."

## Advantages of These Refinements

- **Bounded starvation** — priority boosting gives a hard upper limit on how long any process can be denied fair treatment, regardless of how much interactive load the system is under.
- **Gaming resistance** — cumulative time-at-level accounting removes the incentive (and the ability) for a process to preserve high priority through artificially fragmented, near-instantaneous yields.

## Disadvantages / Trade-offs

- **Tuning complexity** — both the boost interval and the demotion threshold are additional parameters that must be tuned well; too aggressive or too lax settings reintroduce versions of the very problems these refinements are meant to solve.
- **Additional bookkeeping overhead** — tracking cumulative time-at-level per process (rather than a simpler per-round check) requires slightly more state and computation per scheduling decision, though this cost is small relative to the correctness it buys.

## Best Practices

- When designing (or being asked to critique) any scheduling policy, actively look for adversarial or heavy-load edge cases the basic rules might not anticipate — this topic's two scenarios are a template for that kind of thinking, not an exhaustive list.
- Recognize this pattern — "a clean core rule, plus targeted, minimal patches for specific real-world failure modes" — as extremely common throughout systems design generally, not just in scheduling.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Priority boosting and cumulative time accounting solve the same problem." | They solve two distinct problems: priority boosting prevents long-term starvation of low-priority jobs under sustained high-priority load; cumulative accounting prevents a single process from gaming its own priority level through artificially fragmented yields. Both are needed; neither substitutes for the other. |
| "A process that repeatedly yields just before its time slice ends is behaving exactly like a genuinely short, interactive job." | A genuinely short job yields because its actual work is done (or it's waiting on real I/O) — a gaming process yields purely to preserve priority while still consuming nearly a full slice's worth of CPU time every round; cumulative accounting is specifically designed to treat these differently. |

## Interview Questions

1. **Q: What problem does periodic priority boosting solve in MLFQ?**
   A: Long-term starvation — without it, a process demoted to a low priority level could wait indefinitely if there's a constant stream of new, high-priority jobs, since the scheduler always checks higher-priority queues first. Boosting periodically resets every process to the highest priority, guaranteeing a bounded maximum wait.

2. **Q: How can a process "game" the basic MLFQ rules, and how is this fixed?**
   A: By repeatedly using nearly all of its time slice, then yielding briefly (e.g., via a trivial I/O call) just before the slice ends, preserving its high priority indefinitely under the naive "did you yield this round" rule. The fix tracks cumulative CPU time used at the current priority level across all rounds combined, demoting the process once that total crosses a full slice's worth, regardless of how the usage was fragmented.

3. **Q: Why does real-world MLFQ design require more than just the clean core rules from Topic 1?**
   A: Because the basic rules assume yielding behavior always reflects genuine short, interactive work — an assumption that breaks under sustained high interactive load (causing starvation) and under adversarial or accidental gaming (causing unfair priority retention). Robust scheduler design has to explicitly account for both.

## Summary

- Basic MLFQ (Topic 1) can starve long-demoted processes under sustained high-priority load; periodic priority boosting fixes this by giving every process a guaranteed, bounded return to top priority.
- Basic MLFQ can also be gamed by a process that yields just before its time slice ends, repeatedly; tracking cumulative CPU time used at a priority level (rather than resetting on every yield) closes this loophole.
- These refinements exemplify a common systems-design pattern: a clean, elegant core idea, hardened with targeted fixes for specific, realistic failure modes.
- The next topic introduces a fundamentally different scheduling philosophy — lottery and proportional-share scheduling — which reasons about fairness through randomness rather than through priority queues at all.
