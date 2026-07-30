# Interview Questions — Consolidated

All interview questions from every module, collected here for quick revision before an interview. Each entry links back to the module section that explains the full reasoning — don't just memorize the short answer, understand the "why" behind it.

This file grows one module at a time, in the same order as the course itself. Sections for modules not yet written are omitted rather than left as empty placeholders.

---

## Module 01 — Introduction

**Q1. What is an operating system, in one sentence?**
> A layer of software that sits between user programs and physical hardware, making hardware resources (CPU, memory, disk) easier, safer, and more efficient to use than programming them directly.

**Q2. What are the three central problems every operating system solves?**
> Virtualization (making a scarce physical resource look like a private, abundant one to each program), concurrency (correctly coordinating things happening at the same time), and persistence (making data outlive the process — and survive a crash).

**Q3. What is a system call, and why can't a user program just access hardware directly?**
> A system call is a controlled, well-defined entry point through which a user program asks the OS to perform a privileged operation on its behalf (e.g., read a file, create a process). Direct hardware access is blocked because an unrestricted program could corrupt other programs' data, monopolize the CPU, or crash the whole machine — the OS mediates every such request so it can enforce isolation and fairness.

**Q4. What does it mean for the OS to act as a "resource manager"?**
> The OS decides, on behalf of many competing programs, who gets the CPU and for how long, which physical memory a program may use, and which parts of the disk belong to which files — turning contention over shared physical resources into the illusion that each program has its own dedicated machine.

---

## Module 02 — Processes

**Q1. What is a process, and how does it differ from a program?**
> A program is a static file of instructions and data on disk. A process is a specific, live, running instance of that program — its own memory (address space), its own CPU register values (including the program counter), and its own OS-managed resources. Running the same program twice creates two independent processes.

**Q2. What's the difference between the Ready and Blocked process states?**
> Ready means the process could usefully run right now if given a CPU — it's held back purely by scheduling choice. Blocked means the process has nothing useful to do yet regardless of CPU availability, because it's waiting on an external event (most commonly, an I/O operation completing).

**Q3. What is a Process Control Block (PCB), and why is it necessary?**
> A per-process kernel data structure storing everything needed to pause a process and later resume it indistinguishably: its saved register state (including the program counter), its current state, its PID, and its memory/file/scheduling information. Without it, the OS could not correctly restore a paused process's exact execution point.

**Q4. Why is fork() described as "returning twice"?**
> A single call to fork() results in two separate, independent processes existing afterward — the parent and the new child — each resuming from the same point in the code. fork() returns the child's PID in the parent, and 0 in the child, which is how each process tells which one it is.

**Q5. Why does exec() "never return" on success?**
> exec() replaces the calling process's own running code, data, and stack with a new program's, in place, keeping the same PID. On success, the original program's code — including anything after the exec() call — no longer exists in that process, so there is nothing left to return control to. It only returns (with an error) if it fails.

**Q6. What is a zombie process?**
> A process that has finished executing, but whose exit status the OS is still holding onto in its PCB because the parent hasn't yet called wait() to collect it. It's a normal, deliberate, temporary state — it only becomes a practical problem if the parent never calls wait() and zombies accumulate.

**Q7. What happens to a child process if its parent terminates first?**
> It becomes an orphan and is automatically reparented to a designated system process (traditionally init/PID 1), which eventually calls wait() on it, guaranteeing it still gets properly reaped.

**Q8. Why does UNIX use two separate calls, fork() and exec(), instead of one combined "create and run" call?**
> The separation creates a window, after fork() but before exec(), where the child is still running a copy of the parent's own code and can freely reconfigure its own environment (e.g., redirecting file descriptors) before exec() loads a new program — which is exactly how shell features like `command > file` and pipes are implemented, without exec() needing to support every possible setup option directly.

---

## Module 03 — Direct Execution

**Q1. What does "limited direct execution" mean?**
> Running a process's instructions directly on the CPU at full native speed (direct execution), while the OS pre-establishes hardware-enforced restrictions and guaranteed control-transfer points (the "limited" part) so it never loses the ability to intervene or reclaim the CPU.

**Q2. What is a trap table, and when is it configured?**
> A boot-time-registered list mapping each kind of trap (system calls, hardware exceptions, timer interrupts) to a specific, fixed kernel entry point. It's configured once, at boot, while only trusted kernel code runs, and cannot be modified by user-mode code afterward — so a process can request a service but never choose where the hardware actually jumps.

**Q3. Why can't an OS safely rely on processes voluntarily giving back the CPU?**
> Because cooperation isn't guaranteed — a single buggy (e.g., an infinite loop) or malicious process that never traps back into the kernel would freeze the entire machine forever under a purely cooperative model, since nothing would force control back to the OS.

**Q4. What is a timer interrupt, and what guarantee does it provide?**
> A hardware timer, configured by the OS at boot, that forcibly transfers control to the kernel after a fixed interval, regardless of what the currently running process is doing. It guarantees the OS can always reclaim the CPU, even from a process that never voluntarily traps.

**Q5. What's the difference between a mode switch and a context switch?**
> A mode switch (user mode ↔ kernel mode) happens on every single trap or interrupt, even if the OS resumes the same process afterward. A context switch — saving one process's full register state into its PCB and loading a different process's saved state from its own PCB — only happens when the scheduler decides to run a genuinely different process next.

**Q6. Why does each process have a separate kernel stack in addition to its own user-mode stack?**
> Kernel code handling a trap or interrupt on that process's behalf needs a safe, trusted place to store its own temporary state while it works — reusing the process's own (less trusted) user-mode stack for this risks corrupting the very state the kernel is trying to save and restore.

---

## Module 04 — CPU Scheduling Basics

**Q1. What's the difference between turnaround time and response time?**
> Turnaround time is the total time from a job's arrival until it fully completes. Response time is the time from arrival until the job is first given any CPU time at all, even briefly. A policy can score very differently on each — they measure different parts of the experience.

**Q2. What is the convoy effect, and which policy suffers from it?**
> Under FIFO, one or more short jobs can get stuck waiting behind a long-running job, because FIFO is non-preemptive and strictly arrival-ordered — dragging down average turnaround time far more than necessary.

**Q3. How does SJF improve on FIFO, and what's its remaining weakness?**
> SJF runs the shortest available job first instead of the first-arrived job, which is provably optimal for average turnaround time when all jobs are available at once. Because it's non-preemptive, it still suffers a version of the convoy effect when a long job is already running and shorter jobs arrive afterward.

**Q4. How does STCF differ from SJF?**
> STCF adds preemption: whenever a new job arrives, its length is compared against the currently running job's *remaining* time, and the CPU switches to whichever is shorter, even mid-execution. This fully fixes SJF's staggered-arrival weakness and makes STCF provably optimal for average turnaround time.

**Q5. What trade-off does Round Robin make compared to STCF?**
> Round Robin gives every ready job a fixed time slice in a repeating cycle, dramatically improving response time (every job gets attention quickly) at the cost of worse average turnaround time compared to STCF.

**Q6. What happens if Round Robin's time slice is set too short?**
> Context-switching overhead starts to dominate — the CPU spends a significant, wasted fraction of its time performing switches between jobs rather than doing useful work for any of them.

**Q7. What should a scheduler do the instant a running process blocks on I/O?**
> Immediately schedule a different ready process rather than leaving the CPU idle during that process's I/O wait — this is the direct continuation of multiprogramming's goal of keeping the CPU busy, and lets one process's I/O wait overlap with another's useful computation.

---

## Module 05 — CPU Scheduling Advanced

**Q1. What problem does MLFQ solve that STCF cannot in practice?**
> STCF requires knowing a job's total length in advance to make optimal decisions — information a real OS typically doesn't have for a brand-new process. MLFQ observes how a job actually behaves as it runs (does it use its full time slice, or yield early) and adjusts its priority accordingly, requiring no advance knowledge at all.

**Q2. What are MLFQ's core priority-adjustment rules?**
> A new job starts at the highest priority. If it uses its entire time slice without voluntarily yielding, its priority is lowered. If it voluntarily yields before its time slice ends (typically due to I/O), it stays at the same priority level.

**Q3. What problem does periodic priority boosting solve in MLFQ?**
> Long-term starvation — without it, a process demoted to a low priority level could wait indefinitely under sustained high-priority load, since the scheduler always checks higher-priority queues first. Boosting periodically resets every process to the highest priority, bounding the maximum wait.

**Q4. How can a process "game" the basic MLFQ rules, and how is this fixed?**
> By repeatedly using nearly all of its time slice, then yielding briefly just before it ends, preserving high priority indefinitely under the naive "did you yield this round" rule. The fix tracks cumulative CPU time used at the current priority level across all rounds combined, demoting the process once that total crosses a full slice's worth.

**Q5. What does proportional-share scheduling guarantee that turnaround/response-time-oriented policies do not?**
> A specific, chosen percentage share of total CPU time for each process relative to others, rather than optimizing for average completion time or first-scheduling delay.

**Q6. How does lottery scheduling implement proportional share, and what's its main weakness?**
> Each process holds tickets proportional to its intended share; a random winning ticket is drawn at each scheduling decision. Win rates converge to ticket shares only on long-run average — short-term outcomes can deviate meaningfully from the intended proportions purely by chance. Stride scheduling fixes this by achieving the same guarantee deterministically.

**Q7. What is cache affinity, and why does it matter for multi-CPU scheduling?**
> The performance benefit a process gets from running on the same CPU core it ran on before, since that core's cache may still hold data it will reuse. Moving it to a different core forces that core's cache to warm up from scratch, incurring avoidable, slower main-memory accesses.

**Q8. What's the trade-off between single-queue and multi-queue multiprocessor scheduling?**
> Single-queue naturally balances load across cores but requires a shared, lock-protected queue (a scalability bottleneck) and weakens cache affinity. Multi-queue avoids shared-lock contention and preserves cache affinity, but can suffer load imbalance between cores' independent queues, requiring explicit load balancing.

---

## Module 06 — Memory API and Address Spaces

**Q1. What is an address space?**
> The OS's abstraction of memory for a process — a private, contiguous-feeling range of addresses starting at zero that the process uses for all its memory references, entirely independent of where that data physically resides in RAM.

**Q2. What are the three explicit goals memory virtualization must satisfy?**
> Transparency (a process shouldn't be aware its memory is virtualized), efficiency (translation must be fast and its bookkeeping shouldn't consume excessive memory), and protection (no process can access another's memory, or the OS's, without explicit permission).

**Q3. What's the difference between stack and heap memory allocation?**
> Stack allocation is automatic and compiler-managed, tied to the enclosing function call's lifetime. Heap allocation is explicit (via malloc()/free()), programmer-managed, sized at runtime, and persists until explicitly released.

**Q4. Are malloc() and free() themselves system calls?**
> No — they're ordinary library functions managing a heap region the process already has. They only invoke an actual system call (like sbrk or mmap) when the allocator needs to request more memory from the OS, not on every call.

**Q5. What's the difference between a memory leak and a dangling pointer?**
> A leak is memory that's still valid but unreachable and never freed. A dangling pointer is a pointer to memory that has already been freed — using it accesses memory the allocator may have already handed out to a different, unrelated allocation.

**Q6. Why are buffer overflows considered a serious security vulnerability, not just a correctness bug?**
> Writing past a buffer's boundary can overwrite adjacent, unrelated memory — including critical control data like a saved return address — potentially letting a maliciously crafted overflow redirect a program's execution to code of the attacker's choosing.

---

## Module 07 — Address Translation

**Q1. How does base-and-bounds dynamic relocation translate a virtual address to a physical address?**
> It adds the process's base register value to the virtual address to get the physical address, after checking the virtual address is within the range specified by the bounds register — an out-of-range access is rejected rather than completed.

**Q2. Why must base-and-bounds translation happen in hardware rather than OS software?**
> Because it must happen on every single memory access; doing it in software would impose catastrophic per-access overhead. Dedicated hardware (the MMU) performs it near-instantly, with the OS only configuring the base/bounds values once per context switch.

**Q3. What is the main weakness of simple base-and-bounds translation, and how does segmentation fix it?**
> The entire address space, including any unused gap between heap and stack, must occupy one contiguous physical block (internal fragmentation). Segmentation uses multiple independent base/bounds pairs, one per logical segment (Code, Heap, Stack), so physical memory is only allocated for each segment's actual current size.

**Q4. What is external fragmentation, and why does segmentation introduce it?**
> Free physical memory becoming divided into multiple small, non-contiguous chunks, none individually large enough for a new request, even though their total would suffice. It arises because segmentation allows variable-sized segments to be placed and removed independently over time, scattering free space unpredictably.

**Q5. What's the difference between best fit, worst fit, and first fit allocation strategies?**
> Best fit selects the smallest free chunk still large enough for the request, minimizing waste but risking small, awkward leftover slivers. Worst fit selects the largest available chunk, leaving a more useful remainder but depleting large chunks quickly. First fit selects the first chunk encountered that's large enough, trading optimality for much faster search time.

**Q6. What is coalescing, and what problem does it solve?**
> Merging a newly-freed chunk with any physically adjacent free chunks into one larger contiguous chunk. It counteracts fragmentation's tendency to accumulate over time — without it, repeated splitting and freeing would permanently divide the free pool into progressively smaller pieces.

---

## Module 08 — Paging Fundamentals

**Q1. What is the core idea behind paging, and what problem does it solve?**
> Dividing both a process's virtual address space and physical memory into small, fixed-size units (pages and frames). Because every unit is exactly the same size, any free frame can satisfy any page's needs — directly eliminating the external fragmentation that plagued segmentation's variable-sized chunks.

**Q2. Why don't a process's pages need to be stored contiguously in physical memory under paging?**
> Because translation happens independently, per page — each page maps to its own physical frame via the page table, so pages can be scattered arbitrarily across physical memory with no contiguity requirement.

**Q3. How is a virtual address split under paging, and how is each part used?**
> Into a Virtual Page Number (VPN), used to look up the corresponding physical frame number in the page table, and an offset, which identifies the specific byte within that page and passes through unchanged into the physical address.

**Q4. Besides the physical frame number, what else does a page table entry typically store?**
> A valid bit (whether this mapping represents an actually-used page at all), protection bits (read/write/execute permissions), and additional bits like present, reference, and dirty, used by swapping and page-replacement policies.

**Q5. For a 32-bit address space with 4 KB pages and 4-byte PTEs, roughly how large is one process's flat page table?**
> About 4 MB — the address space has 2^20 possible pages (2^32 ÷ 2^12), each needing a 4-byte entry, regardless of how much of that address space the process actually uses.

**Q6. Why does naive paging risk doubling the cost of every memory access?**
> Translating a virtual address first requires reading its page table entry from memory (one access), before the actual intended memory access can proceed (a second access) — turning every logical reference into two physical ones if left unaddressed.

---

## Module 09 — Paging Performance

**Q1. What is a TLB, and what problem does it solve?**
> A small, fast hardware cache of recently used virtual-to-physical translations, on the CPU. It solves paging's "every access costs two memory references" problem by letting the vast majority of accesses (TLB hits) skip the page table lookup entirely.

**Q2. Why does the TLB work well despite being much smaller than the full page table?**
> Locality of reference — programs tend to repeatedly access the same pages (temporal locality) or nearby pages (spatial locality) within any short window, so a small cache of recently used translations satisfies the vast majority of accesses.

**Q3. Why is it dangerous to leave TLB entries unchanged across a context switch?**
> A TLB entry's mapping is specific to the process that generated it; reusing it for a different process (whose identical VPN maps to a different physical frame in its own page table) would incorrectly redirect that process's memory access, violating process isolation.

**Q4. What are the two classic solutions to stale TLB entries across context switches?**
> TLB flushing clears the entire TLB on every switch — simple and always correct, but discards all cached translations. ASIDs tag each TLB entry with its owning process, letting multiple processes' entries safely coexist without a full reset, at the cost of extra hardware support.

**Q5. What specific problem do multi-level page tables solve?**
> A flat page table's size scales with the entire virtual address space, regardless of actual usage. Multi-level tables let entirely unused regions correspond to a single invalid page-directory entry with no second-level table ever allocated, so size scales with actual usage instead — at the cost of more sequential memory reads per TLB miss.

**Q6. What makes an inverted page table "inverted," and what trade-off does it make?**
> It's indexed by physical frame number instead of virtual page number, so its size scales with physical memory (a fixed, system-wide quantity) rather than the sum of every process's virtual address space. The trade-off: translating a (process, VPN) pair requires an auxiliary hash table to avoid an impractically slow linear search.

---

## Module 10 — Beyond Physical Memory

**Q1. What is swap space, and why is disk used for it rather than requiring more RAM?**
> A reserved region of disk used as an overflow area for pages that don't currently fit in physical RAM. Disk is used because it's non-volatile and available in far greater capacity at much lower cost per byte than RAM, trading access speed for holding vastly more data than RAM alone could.

**Q2. What does the present bit indicate, and how does it differ from the valid bit?**
> The present bit indicates whether a legitimate (valid) page is currently resident in RAM or has been swapped out to disk. The valid bit indicates whether a virtual page corresponds to any legitimate part of the address space at all, regardless of current residency.

**Q3. What is a page fault, and how does it differ from an illegal memory access?**
> A hardware-detected exception triggered when a process accesses a valid page that isn't currently resident in RAM — a routine event the OS handles by bringing the page in from swap space. An illegal access (an invalid page or a disallowed operation) has no legitimate page to bring in, and typically results in the process being terminated instead.

**Q4. Walk through what happens after a page fault is triggered.**
> The hardware traps into the kernel's page-fault handler, which locates the page in swap space, ensures a free frame is available (evicting an existing page first if RAM is full), issues a disk read to bring the page into that frame while the process is Blocked, updates the page table entry, and resumes the process by re-executing the original faulting instruction.

**Q5. Why is a page fault handled using the same trap mechanism as a system call?**
> Because it needs the same properties: a safe, hardware-enforced transfer of control to trusted kernel code at a fixed, pre-registered entry point. Reusing the existing trap-table infrastructure avoids inventing a separate control-transfer mechanism for what is, mechanically, just another kind of trap.

---

## Module 11 — Page Replacement Policies

**Q1. What does Belady's optimal replacement policy do, and why can't it be implemented in a real system?**
> It evicts whichever resident page will be used furthest in the future, provably minimizing total page faults. It can't be implemented because it requires knowing the entire future sequence of memory accesses, information that doesn't exist at the moment a real system must make its replacement decision — its real value is as a benchmark for evaluating practical policies.

**Q2. How does FIFO page replacement decide which page to evict, and what's its main weakness?**
> It evicts whichever resident page has been in memory the longest, based purely on arrival order, ignoring actual usage. It's susceptible to Belady's Anomaly — for certain access patterns, adding more physical memory can actually increase total page faults.

**Q3. What principle does LRU rely on, and why is it a reasonable substitute for optimal replacement?**
> Locality of reference — recently-used pages are likely to be used again soon. Since the true future is unknowable, LRU uses the observable recent past as a practical stand-in for predicting near-future access.

**Q4. How does the Clock algorithm approximate LRU cheaply?**
> It maintains a single hardware-set reference bit per page and scans resident pages in a circular order, giving a page with its bit set a "second chance" (clearing the bit and moving on) rather than evicting it immediately — approximating "not recently used" without tracking a fully precise recency order.

**Q5. What is thrashing, and why does it occur regardless of which replacement policy is used?**
> Thrashing occurs when the combined working sets of all running processes exceed available physical memory, causing the system to spend most of its time servicing page faults rather than doing useful work. It happens under any policy because the root problem is insufficient physical memory, not a poor eviction choice.

**Q6. What symptom distinguishes a thrashing system from one that's simply CPU-bound and busy?**
> Low CPU utilization combined with heavy disk I/O activity and most processes sitting Blocked — a thrashing system looks overloaded but its CPU is often comparatively idle, starved for physical memory rather than CPU time.

---

## Module 12 — Concurrency: Threads

**Q1. What is a thread, and how does it differ from a process?**
> A thread is an independent sequence of execution — with its own program counter, registers, and stack — running within a process, sharing that process's single address space with any other threads in it. A process, by contrast, has its own entirely separate, private address space.

**Q2. Give two concrete motivations for using threads instead of separate processes.**
> Exploiting multiple CPU cores for a single logical program (different threads run simultaneously on different cores), and avoiding having the entire program block when one part waits on slow I/O (other threads continue useful work while one is blocked).

**Q3. What do threads of the same process share, and what remains private to each thread?**
> They share code, heap, and global/static data, plus process-wide resources like open files. Each thread keeps its own private program counter, registers, and stack, since each represents an independently-progressing sequence of execution.

**Q4. What is a Thread Control Block (TCB), and how does it relate to a Process Control Block (PCB)?**
> A TCB holds one thread's saved execution state (registers, stack pointer). A process has one PCB (shared, process-wide information) but potentially multiple TCBs, one per thread.

**Q5. Is the relative execution order between two independently-created threads guaranteed?**
> No — the OS scheduler determines actual execution order at runtime, and it can vary between runs of the identical program; either thread's work can happen first unless the program explicitly enforces an order via synchronization.

---

## Module 13 — Locks

**Q1. What is a critical section, and what does mutual exclusion guarantee about it?**
> A critical section is code accessing shared data that must not be interleaved with another thread's access to the same data. Mutual exclusion guarantees at most one thread executes within it at any moment, forcing others to wait.

**Q2. Why can't a correct lock be built using only ordinary load and store instructions?**
> Checking whether the lock is free and marking it as held are separate steps; if interrupted between them, two threads can both see the lock as free and both proceed — the exact race condition the lock was meant to prevent.

**Q3. What does the test-and-set instruction do, and why does its atomicity matter?**
> It atomically reads a memory location's old value and writes a new value as one indivisible hardware step, returning the old value. Its atomicity closes the gap that makes naive check-then-set approaches incorrect.

**Q4. How does compare-and-swap generalize test-and-set?**
> It atomically updates a memory location to a new value only if it currently holds an expected value, reporting success or failure — more flexible than test-and-set, since it supports updating a value only if it hasn't changed since it was last read.

**Q5. Why is spinning particularly wasteful on a single-CPU system?**
> If the lock-holder isn't currently running (because the spinning thread was scheduled instead), the spinning thread's CPU time accomplishes nothing — it actively prevents the lock-holder from getting the CPU time needed to finish and release the lock.

**Q6. What is a two-phase (futex-based) lock?**
> A lock that spins briefly first (cheaply handling the common case where the lock frees up almost immediately), then puts the waiting thread to sleep via a system call if it's still unavailable — combining cheap common-case handling with safe handling of longer waits.

---

## Module 14 — Condition Variables and Semaphores

**Q1. What problem do condition variables solve that a plain lock cannot?**
> A lock only guarantees exclusive access to shared data; it can't express "wait until some specific condition about that data becomes true." Condition variables let a thread sleep until notified the condition may have changed, avoiding wasteful busy-waiting.

**Q2. Why must wait() atomically release the lock as part of going to sleep?**
> If there were a gap between releasing the lock and actually sleeping, another thread could change the condition and signal in that gap, and the waiting thread could miss the wakeup entirely and sleep forever despite the condition already being true.

**Q3. Why should a woken thread re-check its condition in a while loop rather than assume it's true?**
> By the time it reacquires the lock, another thread may have already changed the condition again — re-checking is the only way to safely confirm it still holds before proceeding.

**Q4. What is a semaphore, and how does one initialized to 1 behave like a lock?**
> A synchronization primitive holding an integer value with wait() (decrement, block if negative) and signal() (increment, wake a waiter) operations. Initialized to 1, the first wait() decrements it to 0 and proceeds; any other thread's wait() blocks until a signal() increments it back.

**Q5. Describe the dining philosophers problem and why the naive solution can deadlock.**
> Five philosophers each need both adjacent forks to eat, forks shared circularly. If everyone picks up their left fork simultaneously, every fork is held by someone, and every philosopher waits forever for their right fork, held by an equally stuck neighbor — a complete cycle.

**Q6. How does having one philosopher acquire forks in the opposite order fix the deadlock?**
> It breaks the perfectly symmetric acquisition pattern that allows every fork to simultaneously become someone's "first" fork in a complete cycle, preventing that specific circular waiting condition from forming.

---

## Module 15 — Concurrency Bugs

**Q1. What is an atomicity violation?**
> A sequence of memory accesses intended to execute as one indivisible unit is instead interrupted by another thread's access, because the necessary lock was missing or inconsistently applied — often because one code path (e.g., a rare error branch) doesn't acquire the same lock as the rest.

**Q2. What is an order violation?**
> Code assumes a particular execution order between two threads' operations without actually enforcing it via synchronization, leaving the actual order to depend on however the scheduler happens to interleave them.

**Q3. What are the four necessary conditions for deadlock?**
> Mutual exclusion (a resource can only be held by one thread at a time), hold-and-wait (a thread holds one resource while waiting for another), no preemption (resources can't be forcibly taken away), and circular wait (a cycle of threads each waiting on a resource the next one holds).

**Q4. Why is attacking circular wait via consistent lock ordering the most commonly used deadlock prevention strategy?**
> It requires only a disciplined, agreed-upon rule (acquire locks in a fixed order, everywhere) rather than a fundamental change to how resources behave, unlike attacking mutual exclusion (not always possible) or no preemption (requires complex rollback machinery).

**Q5. What's the fundamental difference between deadlock prevention and deadlock avoidance?**
> Prevention structurally denies one of the four necessary conditions, making deadlock impossible regardless of runtime behavior. Avoidance allows all four to exist but uses runtime checks against declared future resource needs to refuse any request that would leave the system in an unsafe state.

**Q6. How does deadlock detection and recovery differ from both prevention and avoidance?**
> It doesn't try to prevent or avoid deadlock at all — it lets the system run freely, periodically checks for an actual cycle of mutual waiting, and recovers by terminating or rolling back a deadlocked thread when one is found.

---

## Module 16 — Event-Based Concurrency

**Q1. What is an event loop, and how does it differ from the thread-per-task model?**
> A single thread that repeatedly checks which of many registered event sources have work ready, handling each in turn — rather than dedicating a separate OS thread (with its own stack and context-switching overhead) to each concurrent task.

**Q2. Why does the event loop model avoid most shared-state concurrency bugs?**
> Because only one thread ever executes application-level code at a time, there's no possibility of two pieces of application code accessing shared in-memory data simultaneously — the premise behind atomicity violations and most deadlock scenarios doesn't apply.

**Q3. Why is a single blocking call inside an event handler so damaging?**
> The event loop is single-threaded, so any blocking call stalls the entire loop — every other registered event source must wait until it completes, since there's no other thread available to service them.

**Q4. Why does epoll scale better than select()/poll() for servers with many connections?**
> select()/poll() re-scan the entire monitored set on every call, an O(n) cost regardless of readiness. epoll registers descriptors persistently and reports only newly-ready ones, scaling with actual readiness events rather than total monitored count.

---

## Module 17 — I/O Devices and Disks

**Q1. What's the difference between polling and interrupt-driven I/O?**
> Polling repeatedly checks a device's status register in a loop until it reports completion, wasting CPU cycles. Interrupt-driven I/O lets the CPU do other work while waiting, with the device raising a hardware interrupt to notify the kernel once the operation completes.

**Q2. What is DMA, and what specific inefficiency does it remove?**
> Dedicated hardware that transfers data directly between a device and memory without the CPU handling each individual byte, freeing the CPU for other work during the transfer itself.

**Q3. What problem does a device driver solve?**
> It translates generic, uniform requests from the rest of the OS into the specific register operations a particular device's hardware requires, so the rest of the OS never needs device-specific knowledge — new hardware only needs a new driver, not OS-wide changes.

**Q4. What are the three components of hard disk access time?**
> Seek time (moving the disk arm to the correct track), rotational latency (waiting for the platter's spin to bring the correct sector under the head), and transfer time (actually reading/writing once positioned) — seek time and rotational latency are mechanical delays unique to spinning disks.

**Q5. What is SSTF, and what is its main weakness?**
> Shortest Seek Time First services whichever pending request is physically closest to the disk arm's current position, minimizing immediate seek distance — but it can starve requests for physically distant tracks if closer requests keep arriving.

**Q6. How does SCAN fix SSTF's starvation problem?**
> SCAN sweeps the disk arm consistently in one direction, servicing every request it passes, then reverses — guaranteeing every pending request is serviced within at most one full sweep.

---

## Module 18 — RAID

**Q1. How does striping improve performance, and why is RAID 0 worse for reliability than a single disk?**
> Striping spreads data across disks in small chunks, letting requests spanning multiple stripes be serviced in parallel. RAID 0 has zero redundancy, so any one disk's failure corrupts the entire array, and having N disks means N independent chances for such a failure — a higher risk than a single disk.

**Q2. How does RAID 1 achieve fault tolerance, and what does it cost?**
> It keeps a complete, synchronized duplicate of all data on separate disks, so a single disk's failure leaves a full copy intact on the mirror. The cost is capacity: only half of total raw capacity is usable.

**Q3. What is parity, and how does it protect data using less storage than mirroring?**
> A single value computed (typically via XOR) from a set of data blocks, such that any one lost block can be reconstructed from the rest plus the parity. It requires only one extra block's worth of storage regardless of how many blocks it protects.

**Q4. What is RAID 4's write bottleneck, and how does RAID 5 fix it?**
> RAID 4 dedicates one disk entirely to parity, so every write to any data disk also updates that same disk, serializing all writes through it. RAID 5 rotates parity across all disks, distributing updates so writes don't all compete for one disk.

**Q5. What is the usable capacity of a RAID 4/5 array with N disks?**
> (N−1)/N of total raw capacity — one disk's worth of overhead regardless of N, making it more capacity-efficient than mirroring, especially for larger arrays.

---

## Module 19 — Files and Directories

**Q1. What is a file, in the OS sense?**
> The OS's abstraction for a named, persistent sequence of bytes stored on non-volatile storage, which a program can create, read, write, and delete without any awareness of the actual physical disk sectors involved.

**Q2. What does a directory actually contain internally?**
> A list of (name, reference) pairs, mapping human-readable names to either regular files or other sub-directories, stored using the exact same underlying file mechanism as any regular file.

**Q3. What does resolving a full file path require?**
> Walking through each named directory component one level at a time, starting from the root, reading each directory's contents to look up the next component's reference.

**Q4. What is a hard link, and why doesn't deleting one name necessarily delete a file's data?**
> An additional directory entry pointing at the same underlying file, which maintains a reference count of all entries pointing to it. Deleting one entry only decrements the count; the data persists until the count reaches zero.

**Q5. What's the key structural difference between a hard link and a soft link?**
> A hard link directly shares the same underlying file and its reference count. A soft link is a separate file storing just a path, resolved independently each time, and can become a dangling reference if that path stops resolving.

**Q6. What does mounting do?**
> It attaches a separate physical storage device's directory tree onto a specific point within an existing tree, making them appear as one seamless hierarchy, so path resolution can transparently cross physical device boundaries.

---

## Module 20 — File System Implementation

**Q1. What is an inode, and what does it store?**
> A fixed-size, per-file on-disk structure storing metadata (size, owner, permissions, timestamps) and pointers to the actual disk blocks holding the file's data. Every file and directory has exactly one, identified by a unique inode number.

**Q2. What is a bitmap used for in a file system?**
> Tracking which inodes or data blocks are currently free versus in use, one bit per item — a compact, efficient representation made possible by using fixed-size blocks.

**Q3. What does a directory entry actually store on disk?**
> A name paired with an inode number — the file's actual metadata and data lives in the inode itself, not duplicated in the directory entry.

**Q4. Why is deleting a file with existing hard links cheap and non-destructive to the data?**
> Deletion only removes one (name, inode number) directory entry and decrements the target inode's reference count; the data is only actually reclaimed once that count reaches zero, i.e., once every hard link is removed.

**Q5. Why does creating a new file require meaningfully more work than reading an existing one?**
> Reading only looks up already-existing structures. Creating requires genuine allocation: a free inode from the bitmap, an initialized inode, a new directory entry, and eventually data blocks — multiple structures updated, not just looked up.

---

## Module 21 — Fast File Systems and LFS

**Q1. What is the core design principle behind FFS?**
> Placing data likely to be accessed together — a file's inode and its own data, and files within the same directory — physically close together on disk, minimizing seek time given hard disk drives' mechanical realities.

**Q2. What is a cylinder group?**
> A self-contained region of the disk with its own local inodes, bitmaps, and data blocks, keeping a file's inode and its own data physically close together — unlike a naive layout that separates all inodes from all data disk-wide.

**Q3. What is the core idea behind a log-structured file system?**
> Buffering pending writes (inode updates, directory updates, file data) in memory and flushing them all together as one large, sequential write appended to a log, converting many small scattered writes into a single fast, sequential one.

**Q4. Why does LFS need an inode map (imap)?**
> Because LFS never overwrites data in place — every inode update is appended to a new log location, so an inode's physical location changes with every update. The imap tracks each file's current inode location.

**Q5. What is the cleaning problem in LFS?**
> Since old, superseded versions of data are never overwritten, they accumulate as stale space throughout the log. Cleaning periodically scans old segments, distinguishes live from stale blocks, and compacts the live data to reclaim space — an unavoidable consequence of the append-only design.

---

## Module 22 — Crash Consistency

**Q1. What is the crash consistency problem?**
> A file system operation requires multiple separate disk writes; if a crash occurs between some completing and others not, different structures (bitmaps, inodes, directories) can end up disagreeing with each other.

**Q2. What is an orphaned inode?**
> An inode marked "in use" in the bitmap, with valid metadata, but referenced by no directory entry anywhere — arising when a crash occurs after an inode is allocated and initialized but before its directory entry is added.

**Q3. What are fsck's two major limitations?**
> It must scan the entire disk to find and repair inconsistencies, scaling poorly as disk sizes grow, and it can only repair structural inconsistencies, not recover the semantic intent of whatever operation was interrupted.

**Q4. What is the core idea behind journaling?**
> Writing a complete description of an intended update to a dedicated journal before applying those changes to the file system's real structures, so any crash can be resolved by consulting the journal alone rather than inspecting the entire disk.

**Q5. Why does the commit step matter in journaling?**
> It's the critical dividing line: a crash before the commit means the journal entry is incomplete and can be safely discarded; a crash after commit means the entry is guaranteed complete and can be safely replayed to finish the update.

---

## Module 23 — Distributed Systems and Integrity

**Q1. What is silent data corruption, and why is it distinct from RAID's disk-failure protection?**
> Data being altered or misread at the physical level while the read reports success, with no error at all. It's distinct because it can occur on an otherwise fully healthy disk — RAID's redundancy is designed around an entire disk becoming unavailable, not a single block silently returning wrong data.

**Q2. What does a checksum do, and what is its key limitation on its own?**
> It's a value computed from a data block and stored alongside it, recomputed and compared on later reads to detect changes. On its own, it can only detect corruption — it can't recover the original, correct data.

**Q3. How do checksums combine with RAID to provide both detection and correction?**
> A checksum mismatch identifies exactly which block is corrupted; RAID-style redundancy (a mirror or parity reconstruction) then supplies the correct replacement value for that block.

**Q4. What is NFS's core design philosophy, and what trade-off does it make?**
> The server holds the single authoritative copy of data, and clients request nearly every operation from it, keeping minimal local state — strong consistency, at the cost of heavier network load.

**Q5. How does AFS differ from NFS, and what trade-off does it make instead?**
> AFS caches an entire file locally when opened, working from that copy until close, only contacting the server at open/close — much lighter network load, but weaker consistency across simultaneously-open clients.
