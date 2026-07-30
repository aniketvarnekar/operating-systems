# System-Design-Adjacent OS Questions

## Learning Objectives

By the end of this section you should be able to:
- Answer "what happens when you type a command and press Enter" end-to-end, citing the specific modules each step draws from
- Explain how virtual memory concepts inform real database and application design decisions
- Recognize when a broader systems/infrastructure interview question is secretly testing specific OS fundamentals

## Prerequisites

- Modules 01–23, in full — this topic draws on the entire course, synthesized into broader, less module-shaped questions.

## Motivation

Some of the most common OS-adjacent interview questions aren't phrased as "explain concept X" — they're phrased as open-ended systems questions ("what happens when...", "how would you design...") that are secretly testing whether you can recognize and apply specific OS fundamentals within a larger, less structured scenario. This topic covers the most common of these.

## Question 1: "What happens when you type a command in a shell and press Enter?"

This is one of the most common broad systems questions asked in interviews, precisely because a strong answer touches an enormous fraction of this course. A strong, structured answer walks through:

1. **The shell reads your input** (a user-mode process) and parses the command and any arguments.
2. **The shell calls `fork()`** (Module 02, Topic 4) to create a child process — a copy of the shell itself.
3. **In the child, before calling exec()**, the shell sets up any requested I/O redirection or pipes by manipulating file descriptors (Module 02, Topic 7; Module 19, Topic 1) — this is the exact window fork/exec's separation creates.
4. **The child calls `exec()`** (Module 02, Topic 5) to replace itself with the actual requested program, inheriting the file descriptor setup from step 3.
5. **The new program's process runs**, moving between Ready/Running/Blocked (Module 02, Topic 2) as the scheduler (Modules 04–05) grants it CPU time and it issues I/O.
6. **Its memory accesses are translated** via paging (Module 08), with a TLB (Module 09, Topic 1) accelerating the common case, and page faults (Module 10, Topic 2) bringing in any needed data not yet resident.
7. **Any file access** goes through the file abstraction (Module 19), resolved via the file system's on-disk structures (Module 20), potentially involving disk scheduling (Module 17, Topic 4) if the underlying storage is a spinning disk.
8. **The shell (parent) calls `wait()`** (Module 02, Topic 6) to block until the child finishes, then collects its exit status and prints the next prompt.

**What interviewers are actually listening for**: not perfect completeness, but whether you naturally reach for fork/exec/wait as a connected sequence (rather than treating them as isolated facts), and whether you can smoothly bridge from "process creation" into "scheduling" into "memory" into "file systems" as one continuous story rather than four disconnected topics.

## Question 2: "Design a key-value cache — how do you decide what to evict when it's full?"

This question is secretly testing Module 11 (Page Replacement Policies), applied outside the literal context of physical memory pages:

- Start by asking what the workload's access pattern looks like — this maps directly to Module 11, Topic 3's locality-of-reference justification for LRU.
- Propose LRU (or an approximation like Module 11, Topic 3's Clock algorithm, if exact recency tracking is too expensive at scale) as the default, well-justified choice.
- If pushed on "what if LRU isn't ideal for this specific workload," be ready to discuss alternatives (e.g., LFU — least-frequently-used — for workloads where some keys are consistently hot regardless of recency) as a natural extension of the same "past behavior predicts future need" reasoning.

## Question 3: "Our database write throughput drops sharply under heavy concurrent load — how would you investigate?"

This question draws on Modules 13–15 (Concurrency) and Module 17 (I/O):

- First ask whether the drop correlates with lock contention (Module 13) — heavily contended locks serialize access and can dominate wall-clock time under load.
- Check whether the workload is disk-bound and whether writes are sequential or random (Module 17, Topic 3) — random write patterns on spinning-disk-backed storage can be dramatically slower.
- Consider whether deadlock (Module 15, Topics 2–3) or a related transaction-ordering issue is causing timeouts/rollbacks, distinguishing this from a pure performance/contention problem.

**What interviewers are actually listening for**: whether you instinctively separate "is this a concurrency/contention problem" from "is this a physical I/O problem" — two very different root causes this course explicitly taught you to distinguish (Module 13 vs. Module 17), rather than treating "it's slow" as one undifferentiated category.

## Question 4: "Why does our application's memory usage grow slowly over days, even though we're not aware of any obvious bug?"

This tests Module 06, Topic 3 (memory leaks) applied to a realistic, ambiguous production scenario:

- Start by distinguishing a genuine memory leak (allocated memory that's unreachable and never freed) from simply high, legitimate steady-state usage or from OS-level caching that looks like growth but isn't (mention that OS memory reporting can be misleading — cached, reclaimable memory often shows as "used").
- Suggest concrete tools/approaches: memory profiling to find allocation call sites whose allocations are never matched by frees, and checking whether the growth plateaus (bounded, and consistent with legitimate caching) or continues indefinitely (consistent with a genuine leak).

## Best Practices

- For any broad, open-ended systems question, resist the urge to immediately dive into one narrow detail — first sketch the overall shape of your answer (e.g., "I'll walk through process creation, then scheduling, then memory, then file access") so the interviewer can see your structure before you fill in depth.
- When a question doesn't obviously map to a single module, actively ask yourself "which of Virtualization, Concurrency, or Persistence is this really about?" — that framing (this course's own organizing structure) is a fast, reliable way to orient yourself under interview pressure.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "System-design-adjacent OS questions require memorizing a completely separate body of knowledge from the rest of this course." | These questions are almost always testing the exact same fundamentals from Modules 01–23, just asked in a less module-shaped, more open-ended way — the skill being tested is applying known concepts to an unfamiliar framing, not learning new material. |
| "A vague, high-level answer to 'what happens when you type a command' is sufficient." | Interviewers are specifically listening for concrete, connected detail (fork/exec's specific ordering, why wait() matters, how memory translation fits in) — a purely high-level answer signals surface familiarity rather than the depth this course was built to give you. |

## Summary

- "What happens when you type a command and press Enter" is a chance to demonstrate connected knowledge across nearly the entire course — fork/exec/wait, scheduling, memory translation, and file access, told as one continuous story.
- Cache eviction design questions are Module 11's page replacement policies, applied outside the literal context of physical memory.
- Performance-degradation-under-load questions require distinguishing concurrency/contention causes (Modules 13–15) from physical I/O causes (Module 17) — two genuinely different root-cause categories.
- Memory-growth-over-time questions require distinguishing genuine leaks (Module 06, Topic 3) from legitimate steady-state usage or misleading OS-level memory reporting.
