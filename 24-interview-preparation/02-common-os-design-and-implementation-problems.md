# Common OS Design and Implementation Problems

## Learning Objectives

By the end of this section you should be able to:
- Implement (or talk through implementing) the bounded-buffer producer-consumer problem using semaphores
- Explain, from memory, a correct dining philosophers solution
- Implement an LRU cache and explain its relationship to Module 11's page replacement policies
- Reason through a simple scheduler simulation given a set of jobs and a policy

## Prerequisites

- Module 14 (Condition Variables and Semaphores)
- Module 11 (Page Replacement Policies)
- Module 04–05 (CPU Scheduling)

## Motivation

Beyond pure recall questions (Topic 1), many OS interviews ask you to actually **implement** or **simulate** something — often a classic problem this course already covered conceptually. This topic packages four of the most commonly asked implementation-style problems into ready-to-reproduce form, with the reasoning that makes each one make sense rather than just a memorized snippet.

## Problem 1: Bounded-Buffer Producer-Consumer

**The ask**: implement a thread-safe bounded buffer where producer threads add items and consumer threads remove them, without producers overfilling it or consumers removing from an empty buffer.

**The solution**, directly from Module 14, Topic 2 — two counting semaphores plus a mutex for the buffer's own internal bookkeeping:

```c
sem_t empty, full;
mutex_t mutex;
sem_init(&empty, BUFFER_SIZE);  // starts as fully empty (all slots available)
sem_init(&full, 0);              // starts with zero filled slots

void producer() {
    sem_wait(&empty);      // wait for an empty slot
    lock(&mutex);
    add_item_to_buffer();
    unlock(&mutex);
    sem_post(&full);        // signal one more filled slot
}

void consumer() {
    sem_wait(&full);        // wait for a filled slot
    lock(&mutex);
    remove_item_from_buffer();
    unlock(&mutex);
    sem_post(&empty);       // signal one more empty slot
}
```

**The reasoning an interviewer wants to hear**: `empty` and `full` handle the *condition-based* waiting (don't overfill, don't underfill) — exactly Module 14, Topic 2's insight that a semaphore's integer value can directly represent a resource count. The separate `mutex` protects the buffer's own internal data structure from simultaneous modification — a plain mutual-exclusion concern (Module 13), independent of the counting concern. Conflating these two semaphores' roles with the mutex's role is the most common mistake here.

## Problem 2: Dining Philosophers

**The ask**: implement (or describe) a deadlock-free solution to the dining philosophers problem.

**The solution**, directly from Module 14, Topic 3 — break the acquisition-order symmetry for one philosopher:

```c
void philosopher(int i) {
    if (i == NUM_PHILOSOPHERS - 1) {
        sem_wait(&fork[(i+1) % NUM_PHILOSOPHERS]);  // RIGHT first (the one exception)
        sem_wait(&fork[i]);                          // then LEFT
    } else {
        sem_wait(&fork[i]);                          // LEFT first
        sem_wait(&fork[(i+1) % NUM_PHILOSOPHERS]);   // then RIGHT
    }
    eat();
    sem_post(&fork[i]);
    sem_post(&fork[(i+1) % NUM_PHILOSOPHERS]);
}
```

**The reasoning an interviewer wants to hear**: map this directly onto Module 15, Topic 2's four necessary conditions for deadlock. This fix specifically denies **circular wait** — with one philosopher's acquisition order reversed, no complete cycle of "everyone holding one fork, waiting on the next" can form. Being able to state *which* of the four conditions a given fix denies (rather than just "it works") is what separates a strong answer from a merely correct one.

## Problem 3: Implementing an LRU Cache

**The ask**: implement a cache with a fixed capacity that evicts the least-recently-used item when full.

**The solution**: a hash map (for O(1) key lookup) combined with a doubly-linked list (for O(1) reordering of recency), a classic combination:

```
 get(key):
     if key not in map: return NOT_FOUND
     move key's node to the FRONT of the list (most recently used)
     return its value

 put(key, value):
     if key already in map: update value, move its node to the FRONT
     else:
         if cache is at capacity:
             remove the node at the BACK of the list (least recently used)
             remove its entry from the map
         insert a new node at the FRONT; add it to the map
```

**The reasoning an interviewer wants to hear**: this is precisely Module 11, Topic 3's LRU replacement policy, implemented as a general-purpose data structure rather than applied specifically to physical page frames. Explicitly drawing this connection — "this is the same eviction principle as LRU page replacement, just applied to an arbitrary cache instead of physical memory frames" — signals genuine understanding rather than memorized boilerplate. Also be ready to explain why a hash map alone isn't sufficient (no efficient way to track recency ordering) and why a linked list alone isn't sufficient (no O(1) key lookup) — the combination is what gives both operations O(1) time.

## Problem 4: Simulating a Simple Scheduler

**The ask**: given a list of jobs (arrival time, burst length) and a scheduling policy, compute each job's completion time, turnaround time, and/or response time.

**The approach**, directly from Module 04:

1. Maintain a "current time" pointer and a ready queue of arrived-but-not-yet-completed jobs.
2. At each decision point, apply the policy's specific rule to choose the next job from the ready queue:
   - **FIFO**: whichever arrived first among those currently ready.
   - **SJF**: whichever has the shortest total burst length among those currently ready (non-preemptive — once chosen, it runs to completion).
   - **STCF**: re-evaluate at every arrival; whichever has the shortest **remaining** time wins, potentially preempting a running job.
   - **Round Robin**: cycle through the ready queue, giving each job exactly one time-slice-sized chunk before moving to the next.
3. Track each job's completion time as it finishes; compute turnaround time (`completion − arrival`) and response time (`first_run − arrival`) directly from the trace.

**The reasoning an interviewer wants to hear**: work through a concrete example out loud, narrating the ready queue's state at each step — interviewers are testing whether you can *mechanically trace* the policy correctly, not just recite its definition. Always double-check whether the policy is preemptive (STCF, Round Robin) or not (FIFO, SJF) before starting the trace, since that single detail changes the entire simulation's structure.

## Best Practices

- For any "implement X" question, first state out loud which underlying OS concept X is really testing (a semaphore's dual role as lock/condition; deadlock's four conditions; a replacement policy as a general data structure pattern) before writing any code — this signals structured thinking rather than pattern-matching from memory.
- Practice tracing the scheduler simulation by hand on paper for at least one small example of each policy (FIFO, SJF, STCF, Round Robin) before an interview — the mechanical trace is what's actually being tested, not the policy's one-sentence definition.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The mutex in the producer-consumer solution is redundant since the semaphores already handle synchronization." | The empty/full semaphores handle condition-based waiting (is there room/an item); the mutex separately protects the buffer's own internal data structure from simultaneous modification — removing it can cause corruption even though the semaphores are correctly preventing overfill/underflow. |
| "An LRU cache just needs a hash map; a linked list isn't necessary." | A hash map alone gives fast key lookup but no efficient way to track or update recency ordering — the linked list is what provides O(1) "move to front" and "remove from back" operations; without it, finding the least-recently-used item would require an O(n) scan. |

## Summary

- The bounded-buffer problem combines two counting semaphores (condition-based waiting) with a mutex (mutual exclusion) — conflating their distinct roles is the most common mistake.
- A correct dining philosophers solution denies deadlock's circular-wait condition specifically, by breaking acquisition-order symmetry for at least one participant.
- An LRU cache is Module 11's LRU replacement policy generalized into a hash-map-plus-linked-list data structure, giving O(1) get/put with correct eviction.
- Simulating a scheduler requires correctly tracing the ready queue's state at each decision point, with preemptive policies (STCF, Round Robin) requiring re-evaluation at every arrival, unlike non-preemptive ones (FIFO, SJF).
