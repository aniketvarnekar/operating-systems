# Module 04 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Scheduling Metrics** — turnaround time, response time, and fairness, with precise formulas
- [x] **First-In, First-Out (FIFO) Scheduling** — the simplest policy, and the convoy effect it exposes
- [x] **Shortest Job First (SJF)** — fixing the convoy effect for simultaneous arrivals, and its staggered-arrival weakness
- [x] **Shortest Time-to-Completion First (STCF)** — adding preemption, provable turnaround-time optimality, and its response-time cost
- [x] **Round Robin (RR)** — optimizing for response time via time slices, and the time-slice-length trade-off
- [x] **Incorporating I/O** — modeling jobs as CPU/I/O bursts, and overlapping I/O with computation to maximize CPU utilization

## The Big Picture: A Comparison Table

| Policy | Preemptive? | Needs job length? | Turnaround time | Response time | Key weakness |
|---|---|---|---|---|---|
| FIFO | No | No | Poor (convoy effect) | Poor | Long jobs block short ones |
| SJF | No | Yes | Optimal (simultaneous arrivals only) | Poor | Fails under staggered arrivals |
| STCF | Yes | Yes | Optimal (always) | Poor | Non-shortest jobs can wait a long time for first turn |
| Round Robin | Yes | No | Worse than STCF | Excellent | Time-slice length must be carefully tuned |

This table is the real takeaway of the module: **there is no universally "best" scheduler** — each policy is the direct, deliberate answer to a specific flaw in the one before it, and each fix trades away something else. FIFO → SJF fixes the convoy effect for simultaneous arrivals. SJF → STCF fixes staggered arrivals via preemption. STCF → Round Robin trades optimal turnaround time for dramatically better response time. Topic 6 then showed that real jobs aren't monolithic at all — they're sequences of CPU/I/O bursts, and a good scheduler exploits the gaps created by I/O waits to keep the CPU busy with other work.

## Practical Connections

- **Why a big batch script (like a video render) doesn't feel like it's "hogging" your computer, even while your mouse and keyboard still feel responsive** — modern OS schedulers blend Round-Robin-like time-slicing for interactive fairness with burst-aware, STCF-like preference for short-running foreground tasks.
- **Why increasing "priority" of a process on your OS (`nice` on Linux/macOS, Task Manager priority on Windows) actually does something** — it biases the scheduler's decisions in exactly the direction this module discussed: more or less preferential access to the CPU relative to other ready processes.
- **Why a system that's genuinely out of spare CPU capacity feels sluggish across *every* app at once, not just one** — once there are more ready processes needing CPU time than the scheduler can serve promptly, average response time degrades for everyone, regardless of which specific policy is in use.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Turnaround time vs. response time | Turnaround time is arrival-to-completion; response time is arrival-to-first-scheduled. A policy can excel at one while performing poorly on the other. |
| SJF vs. STCF | SJF is non-preemptive (chooses once, per job); STCF is preemptive (re-evaluates whenever a new job arrives, comparing against the running job's *remaining* time). |
| STCF vs. Round Robin | STCF optimizes for turnaround time at the cost of response time; Round Robin optimizes for response time at the cost of turnaround time — opposite ends of the same trade-off. |
| Whole-job scheduling vs. burst-aware scheduling | Whole-job scheduling (Topics 2–5's examples) treats a job as one solid block of CPU time; burst-aware scheduling (Topic 6) makes a fresh decision at each CPU-burst boundary, accounting for I/O waits in between. |

## What's Next

This module covered the classic, foundational scheduling algorithms — but all of them share a hidden assumption: the scheduler knows (or can estimate) each job's length in advance, and none of them explicitly reason about a process's priority *changing* based on its own observed behavior over time. **Module 05 — CPU Scheduling Advanced** removes that assumption, introducing the Multi-Level Feedback Queue (which learns a process's likely behavior from its history rather than requiring advance knowledge), lottery/proportional-share scheduling (a fundamentally different, probabilistic approach to fairness), and multi-CPU scheduling (what changes once there's more than one physical core to schedule across).
