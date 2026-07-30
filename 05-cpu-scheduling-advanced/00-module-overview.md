# Module 05 — CPU Scheduling Advanced

## Module Goal

By the end of this module, you will understand **three scheduling ideas that go beyond Module 04's classic algorithms**: a scheduler that learns a process's likely behavior from history instead of requiring job lengths to be known in advance (MLFQ), a fundamentally different, probability-based approach to fairness (lottery/proportional-share scheduling), and what changes once there's more than one physical CPU core to schedule across at all.

## Topics Covered in This Module

1. **[The Multi-Level Feedback Queue (MLFQ)](01-the-multi-level-feedback-queue.md)** — Approximating STCF's good turnaround time and Round Robin's good response time simultaneously, without ever needing to know a job's length in advance.
2. **[MLFQ Refinements and Gaming Resistance](02-mlfq-refinements-and-gaming-resistance.md)** — The priority-boost fix for long-term starvation, and how the basic rules must be hardened against processes that deliberately exploit them.
3. **[Lottery and Proportional-Share Scheduling](03-lottery-and-proportional-share-scheduling.md)** — A completely different philosophy: guaranteeing each process a proportional *share* of the CPU via randomness, instead of reasoning about turnaround or response time directly.
4. **[Multi-CPU Scheduling](04-multi-cpu-scheduling.md)** — Cache affinity, single-queue vs. multi-queue designs, and load balancing, once there's more than one core to schedule across.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- Module 04 in full — this module assumes fluency with turnaround time, response time, and the FIFO/SJF/STCF/Round Robin trade-offs it builds directly on top of.

## How to Study This Module

Read in order. Topics 1–2 form one continuous idea (MLFQ): Topic 1 gives you the elegant core rules and shows how they approximate Module 04's best policies without their key weakness (needing job length in advance); Topic 2 shows the real, non-obvious problems that appear once you imagine a genuinely adversarial or long-running-but-interactive process trying to exploit those rules, and how real schedulers close those loopholes. Topic 3 is a deliberate change of philosophy — instead of reasoning about turnaround/response time directly, it reasons about proportional *fairness* using randomness, which is a surprisingly different and elegant way to think about the same underlying resource-sharing problem. Topic 4 then relaxes an assumption every single topic in Modules 04–05 up to this point has quietly made — that there's exactly one CPU to schedule — and shows what breaks (and what new opportunities appear) once there's more than one.
