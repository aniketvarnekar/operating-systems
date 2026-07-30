# Module 16 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The Event Loop and select()/poll()** — one thread managing many I/O sources via readiness checks, and why this avoids most shared-state concurrency bugs by construction
- [x] **The Blocking Call Problem and epoll** — why a single blocking call stalls the entire loop, strategies to avoid it, and epoll's scaling improvement over select/poll

## The Big Picture

This module presented a genuinely different answer to Module 12's original question ("how do we handle many things at once?") — one thread and an event loop, instead of many OS threads. Its trade-off is the mirror image of the thread-based model's: it trades away Module 15's entire shared-state bug category for a new, absolute rule of its own.

```
   Module 12–15: MANY THREADS, one shared address space
       + = exploit multiple cores, natural blocking-call handling
       − = needs locks (Module 13), condition variables (Module 14),
           and constant vigilance against Module 15's bug categories

   Module 16: ONE THREAD, an event loop
       + = no locks needed at all for its own logic — most shared-state
           bugs are structurally impossible
       − = a SINGLE blocking call anywhere stalls EVERYTHING;
           can't exploit multiple cores on its own
```

## Practical Connections

- **Why Node.js and similar single-threaded, event-driven runtimes emphasize "never block the event loop" as their single most important rule** — this is Topic 2's blocking-call problem, stated as the load-bearing design constraint of an entire popular platform.
- **Why high-performance web servers (nginx, and similar) are built around epoll (or an OS-specific equivalent) rather than plain select/poll** — Topic 2's scaling argument directly explains this widely-adopted, real-world architectural choice.
- **Why some systems combine both models** — a small number of event-loop processes/threads (one roughly per CPU core, to address Topic 1's "can't exploit multiple cores alone" limitation), each handling many connections internally via its own event loop — getting a practical blend of both models' advantages.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Thread-based concurrency (Modules 12–15) vs. event-based concurrency (this module) | Thread-based uses many OS threads sharing an address space, needing locks/condition variables to stay correct. Event-based uses one thread and a readiness-checking loop, sidestepping most shared-state bugs but requiring every operation to be non-blocking. |
| select()/poll() vs. epoll | select()/poll() re-scan the entire monitored set on every call (O(n) per call). epoll registers descriptors persistently and reports only newly-ready ones, scaling with actual readiness events instead of total monitored count. |
| A blocking call's effect in a threaded model vs. an event-loop model | In a threaded model, one thread blocking (Module 02, Topic 2) doesn't stop other threads from running. In a single-threaded event loop, any blocking call stalls every other managed task, since there's no other thread to service them. |

## What's Next

Modules 12–16 completed Concurrency in full: threads, locks, condition variables/semaphores, the bugs that arise from misusing them, and an alternative single-threaded model that sidesteps most of those bugs at the cost of a strict non-blocking discipline. **Module 17 — I/O Devices and Disks** begins the course's third and final major theme, Persistence (introduced conceptually in Module 01, Topic 5): the physical devices data is stored on, the canonical protocol the OS uses to communicate with them, and hard disk drive mechanics — the hardware foundation everything in Modules 18–23 builds on.
