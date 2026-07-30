# Core Conceptual Questions — Ranked & Synthesized

## Learning Objectives

- Recall, from memory and in your own words, the highest-signal OS conceptual questions across the entire course
- Answer each in interview-appropriate form: a tight, complete answer in 30–60 seconds, not a five-minute lecture
- Know exactly where to go for full depth if an interviewer pushes further

## Prerequisites

Modules 01–23, in full — this topic is pure retrieval practice over material you've already learned.

## Motivation

`interview-questions.md` at the course root already gives you every Q&A this course has produced, in module order — comprehensive, but not **prioritized**. In a real interview, some questions come up constantly across companies and interview styles (virtual memory, deadlock, scheduling, crash consistency), while others are rarer, deeper cuts. This topic re-organizes the course's highest-signal questions **by theme**, roughly in the order interviewers actually reach for them, so your review time goes to what matters most first.

**How to use this list:** cover the answer, say the question out loud, answer it out loud in your own words in under a minute, *then* check the model answer. Reading the answer first feels productive but builds almost no real interview recall.

## Theme: Virtualization — Processes and Scheduling

**Q1. What is a process, and how does it differ from a program?**
> A program is a static file on disk; a process is a specific, live, running instance of it — its own memory, registers, and program counter. Running the same program twice creates two independent processes. (Module 02, Topic 1)

**Q2. Why does fork() "return twice"?**
> fork() creates a child process as a copy of the parent; afterward, two independent processes exist, both resuming from the same point — fork() returns the child's PID in the parent and 0 in the child. (Module 02, Topic 4)

**Q3. Why are fork() and exec() separate system calls in UNIX?**
> The separation creates a window, after fork() but before exec(), where the child still runs the parent's code and can reconfigure its own environment (e.g., redirecting file descriptors) before exec() loads a new program — enabling shell features like `>` and `|` without exec() needing to support every possible setup option. (Module 02, Topic 7)

**Q4. What is limited direct execution, and why is it "limited"?**
> Running a process's instructions directly on the CPU for speed, while the OS pre-establishes hardware-enforced boundaries (restricted operations via a trap table, and a timer interrupt) so it never loses the ability to intervene or reclaim the CPU. (Module 03, Topic 1)

**Q5. Why can't an OS rely on processes voluntarily giving back the CPU?**
> Cooperation isn't guaranteed — an infinite-looping or malicious process would freeze the machine forever under a purely cooperative model. A hardware timer interrupt forces control back to the kernel regardless of what the process is doing. (Module 03, Topic 3)

**Q6. What's the difference between SJF, STCF, and Round Robin?**
> SJF runs the shortest job first but is non-preemptive, so it can't react to a shorter job arriving after a longer one starts. STCF adds preemption, fixing that and becoming provably optimal for turnaround time. Round Robin instead optimizes for response time via short time slices, at a real cost to turnaround time. (Module 04, Topics 3–5)

**Q7. How does MLFQ approximate STCF without knowing job length in advance?**
> It observes behavior instead of requiring foreknowledge: new jobs start at the highest priority; using a full time slice without yielding demotes a job; yielding early (typical of short/interactive work) keeps it at high priority. (Module 05, Topic 1)

## Theme: Virtualization — Memory and Paging

**Q8. What are the three goals of memory virtualization?**
> Transparency (a process shouldn't know its memory is virtualized), efficiency (translation must be fast, bookkeeping shouldn't be excessive), and protection (no process can access another's memory without permission). (Module 06, Topic 1)

**Q9. Why does paging avoid the fragmentation problems of segmentation?**
> Paging uses fixed-size pages and frames — since every chunk is identical, any free frame satisfies any page's request, with no size-matching decision, eliminating external fragmentation. Segmentation's variable-sized regions inevitably scatter free space into unusable fragments. (Module 08, Topic 1)

**Q10. What problem does the TLB solve, and why does it work?**
> It caches recently used translations so most memory accesses skip the full page table lookup, solving paging's "every access costs two memory references" problem. It works because of locality of reference — programs cluster their accesses around a small set of pages at any given time. (Module 09, Topic 1)

**Q11. Why do multi-level page tables save space compared to a flat table?**
> A flat table's size scales with the entire address space, regardless of usage. A multi-level table lets entirely unused regions correspond to a single invalid page-directory entry with no second-level table allocated at all, so size scales with actual usage instead. (Module 09, Topic 3)

**Q12. What is a page fault, and how does the OS handle it?**
> A hardware trap triggered when a process accesses a valid page that isn't currently resident in RAM. The kernel locates the page in swap space, ensures a free frame is available (evicting one if needed), reads the page in while the process blocks, updates the page table, and resumes the faulting instruction. (Module 10, Topic 2)

**Q13. Why is Belady's optimal replacement policy unimplementable, and what is it used for instead?**
> It requires knowing the entire future sequence of memory accesses — information unavailable at decision time. Its real value is as a benchmark: real policies are judged by how closely they approach it. (Module 11, Topic 1)

**Q14. What is thrashing, and why can't a better replacement policy fix it?**
> Thrashing occurs when combined working-set demand across all running processes exceeds physical memory, causing constant page faults regardless of policy — the root problem is insufficient capacity, not a poor eviction choice. The fix is reducing the degree of multiprogramming or adding memory. (Module 11, Topic 4)

## Theme: Concurrency

**Q15. Why can't a correct lock be built from ordinary load/store instructions?**
> Checking whether a lock is free and marking it held are separate steps; if interrupted between them, two threads can both see it as free and both proceed. Hardware atomic instructions (test-and-set, compare-and-swap) close this gap. (Module 13, Topic 2)

**Q16. Why must a thread calling wait() on a condition variable hold the associated lock?**
> So releasing the lock and going to sleep happens as one atomic step — otherwise another thread could change the condition and signal in the gap, and the waiting thread could miss the wakeup entirely. (Module 14, Topic 1)

**Q17. What are the four necessary conditions for deadlock?**
> Mutual exclusion, hold-and-wait, no preemption, and circular wait — all four must hold simultaneously; denying any one makes deadlock impossible. (Module 15, Topic 2)

**Q18. Why is consistent lock ordering the most commonly used deadlock prevention strategy?**
> It requires only a disciplined, agreed-upon rule rather than a fundamental change to how resources behave, unlike attacking mutual exclusion (often impossible) or no preemption (requires complex rollback). (Module 15, Topic 2)

**Q19. What's the difference between an atomicity violation and an order violation?**
> An atomicity violation breaks an intended indivisible sequence via a missing or inconsistent lock. An order violation assumes an execution order between threads that isn't actually enforced by synchronization. Neither involves a waiting cycle, unlike deadlock. (Module 15, Topic 1)

**Q20. Why does event-based concurrency avoid most shared-state concurrency bugs?**
> Only one thread ever executes application code at a time, so there's no possibility of two pieces of code accessing shared data simultaneously — at the cost of a single blocking call anywhere stalling the entire loop. (Module 16, Topics 1–2)

## Theme: Persistence

**Q21. Why is sequential disk access dramatically faster than random access on a hard disk drive?**
> Sequential access minimizes seek time (arm movement) and rotational latency by keeping requests physically close together; random access incurs a fresh, often substantial seek and rotational wait for nearly every request. (Module 17, Topic 3)

**Q22. Why is RAID 0 worse for reliability than a single disk, not just "no better"?**
> Data is striped with zero redundancy, so any one disk's failure corrupts the entire array; with N disks, there are N independent chances for such a failure, a higher risk than one disk alone. (Module 18, Topic 1)

**Q23. What is an inode, and how does it relate to a directory entry?**
> An inode is a per-file structure storing metadata and pointers to data blocks. A directory entry is just a (name, inode number) pair — a hard link is simply a second entry using the same inode number. (Module 20, Topics 1–2)

**Q24. What is the crash consistency problem, and how does journaling solve it better than fsck?**
> A file system operation requires multiple separate disk writes; a crash between them can leave structures inconsistent. fsck detects and repairs this by scanning the entire disk afterward — slow and scales poorly. Journaling records intent in a small journal before acting, so recovery only needs to replay or discard journal entries, without a full-disk scan. (Module 22, Topics 1–2)

## Best Practices for Using This List

- Time yourself: a strong 30–60 second answer beats a rambling three-minute one — interviewers read excessive length as uncertainty, not thoroughness.
- Always be ready to go one level deeper than the model answer if asked "why" again — every question here links back to a full module topic for exactly that purpose.
- Practice explaining these out loud, not just recalling them silently — the muscle interviews actually test is verbal, real-time explanation, not passive recognition.

## Common Mistakes

- Reciting a memorized definition without connecting it to why it matters — interviewers consistently rate "explains the reasoning" answers higher than "recites the fact" answers, even when both are technically correct.
- Answering with excessive scope-creep — turning "what's the difference between a hard link and a soft link" into an unprompted tour of the entire file system module. Answer what was asked; let the interviewer ask a follow-up if they want more.
- Freezing on a question you don't remember instead of reasoning toward it out loud — interviewers routinely value visible, structured reasoning under uncertainty over a memorized-but-silent struggle.

## Summary

- This topic re-organized the course's highest-signal conceptual questions by cross-cutting theme (Processes/Scheduling, Memory/Paging, Concurrency, Persistence) rather than by module order, prioritizing what interviewers ask most.
- Every answer is deliberately interview-length (30–60 seconds spoken), with a pointer back to the full module topic for further depth.
- The full, unranked, module-ordered Q&A set remains available in the course root's `interview-questions.md` for exhaustive review.
