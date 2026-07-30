# Networked File Systems: NFS and AFS

## Learning Objectives

By the end of this section you should be able to:
- Explain what a networked file system is and why it needs to extend Module 19's file abstraction across a network
- Explain NFS's server-driven, minimal-client-caching design and its trade-offs
- Explain AFS's client-caching design and how it improves on NFS's network-load trade-off

## Prerequisites

- Module 19 in full (Files and Directories)
- Module 01, Topic 4 (Concurrency) — the caching-consistency questions here echo similar coordination concerns

## Motivation

Every module since Module 17 assumed "the disk" is physically attached to the machine using it. This topic relaxes that assumption entirely: what happens when a file's actual physical storage lives on a completely different, remote machine, accessed only over a network — while still, ideally, looking and behaving exactly like an ordinary local file to the program using it?

## Problem Statement

Module 19, Topic 1 established the file abstraction specifically to hide *physical disk sector* details from programs. Could that same abstraction be extended one level further, to also hide the fact that a file's data might not even be on the *same machine* at all — letting a program `open()`, `read()`, and `write()` a remote file using the exact same API as a local one, with the network communication happening entirely transparently underneath?

## Concept

### Networked File Systems

> A **networked (or distributed) file system** lets a client machine access files that physically reside on a separate, remote server machine, using the same file and directory API (Module 19) a program would use for purely local files — the client-server network communication required to actually fetch and modify the remote data is handled transparently by the networked file system's own client-side and server-side software.

This directly extends Module 06, Topic 1's transparency goal (and Module 19, Topic 3's mounting mechanism, which is often literally how a remote file system's directory tree gets attached into a client's local directory hierarchy) across an entire network boundary.

### NFS: Server-Driven, Minimal Client State

> **NFS (Network File System)** was designed around a deliberately simple philosophy: the **server** holds the authoritative, single copy of all file data, and clients send requests (read this block, write this block) to the server for essentially every operation, keeping very little of their own persistent local state about a file's contents.

- **Advantage**: simplicity, and strong consistency — since the server is the single source of truth for every read and write, multiple clients accessing the same file generally see a consistent, up-to-date view, because there's no significant client-side caching of stale data to worry about.
- **Disadvantage**: **heavy network load** — because clients rely on the server for nearly every operation rather than caching much locally, network traffic between clients and the server can be substantial, especially for files accessed repeatedly by the same client.

### AFS: Client-Side Caching for Reduced Network Load

> **AFS (Andrew File System)** takes a different approach: when a client opens a file, it **caches an entire copy of that file locally**, on the client's own disk, and works with that local copy for subsequent reads/writes — only communicating with the server again when the file is first opened (to fetch the current copy) and when it's closed (to send back any changes).

- **Advantage**: **dramatically reduced network load** for repeated access to the same file by the same client — once cached, reads and writes hit the client's own local disk, not the network, until the file is closed.
- **Disadvantage**: **weaker consistency guarantees** — if two different clients have the same file open simultaneously, each working from their own locally cached copy, changes made by one client are not visible to the other until the first client closes the file (and the second client subsequently reopens it) — a real, deliberate trade-off compared to NFS's tighter, server-driven consistency.

### The Underlying Trade-off: Network Load vs. Consistency

This is a direct, concrete instance of a general distributed-systems trade-off: **caching data locally reduces network communication and improves performance for repeated access, but makes it harder to guarantee that every client always sees the most up-to-date, consistent view of shared data** — the more aggressively a client caches, the more network load is saved, but the more potential there is for different clients to temporarily see different, inconsistent versions of the same file.

## Internal Working (Preview)

```
   NFS: server-driven, minimal client caching

   Client A ──(request: read block 5)──► Server (authoritative data)
   Client A ◄─(here's block 5's data)──── Server
   Client B ──(request: write block 5)──► Server (updates authoritative copy)

   → every operation round-trips to the server — HEAVY network load,
     but STRONG consistency (server is always the single source of truth)


   AFS: client-side caching

   Client A: open(file) ──► Server sends ENTIRE file  ──► Client A caches it LOCALLY
   Client A: read/write  ──► happens LOCALLY, no network traffic at all
   Client A: close(file) ──► Client A sends CHANGES back to Server

   Meanwhile, Client B: open(SAME file) ──► gets whatever version the
     Server currently has — may NOT reflect Client A's still-open,
     uncommitted local changes

   → LOW network load for repeated access, but WEAKER consistency
     across simultaneously-open clients
```

## Real-World Analogy

NFS is like a shared, single physical company ledger kept at a central office, where every single employee (client), no matter how far away, has to call the office and ask a clerk to look something up or record a change for every single transaction — slow and creates a lot of phone traffic, but everyone is always working from the exact same, immediately up-to-date ledger. AFS is like instead handing each employee a full photocopy of the relevant pages when they start their work session, letting them mark it up freely at their own desk without calling the office at all, and only mailing their marked-up copy back to the office once they're completely done — dramatically less phone traffic during the work session, but if two employees happened to take copies of the same pages at the same time, neither will see the other's edits until they've both finished and mailed their copies back.

## Why Both Design Philosophies Are Legitimate

Neither NFS's nor AFS's approach is universally "better" — they represent different, deliberate points on the same network-load-versus-consistency trade-off spectrum this topic identified. NFS suits workloads where strong, immediate consistency across multiple simultaneous clients matters most, and where network capacity is ample. AFS suits workloads with more repeated, sustained access per client (where caching pays off enormously) and where perfectly immediate cross-client consistency is less critical than reducing network load — this same fundamental trade-off (cache aggressively for performance, vs. minimize caching for consistency) reappears constantly throughout distributed systems design generally, well beyond file systems specifically.

## Advantages and Disadvantages Summary

- **NFS**: strong consistency, simple model — but heavier network load, especially for repeated access.
- **AFS**: dramatically reduced network load for repeated access — but weaker consistency guarantees when multiple clients access the same file concurrently.

## Best Practices

- When evaluating a networked file system (or any distributed caching system) for a given workload, explicitly ask where it sits on the network-load-versus-consistency spectrum, and whether that trade-off matches the workload's actual access patterns (many small, scattered accesses favoring less caching; repeated, sustained access by the same client favoring more caching).
- Recognize this trade-off as a recurring, general theme in distributed systems — not a quirk specific to NFS and AFS alone.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "AFS's local caching means it provides the exact same consistency guarantees as NFS, just faster." | AFS's caching model means changes made by one client aren't visible to another client's already-open copy of the same file until the first client closes it — a genuinely weaker consistency guarantee than NFS's server-driven, per-operation model, not merely a performance difference. |
| "NFS and AFS are simply two implementations of the identical design, just from different companies." | They represent fundamentally different design philosophies regarding client-side caching — NFS minimizes client state and relies on the server for nearly every operation, while AFS caches entire files locally and only synchronizes with the server on open/close — a structural difference with real consistency and performance trade-offs, not just an implementation detail. |

## Interview Questions

1. **Q: What is a networked file system, and what abstraction does it extend across a network?**
   A: A system that lets a client machine access files physically residing on a remote server using the same file and directory API as local files, extending the file abstraction's transparency goal (Module 19, Topic 1) across a network boundary.

2. **Q: What is NFS's core design philosophy, and what trade-off does it make?**
   A: The server holds the single authoritative copy of all data, and clients send requests to the server for essentially every operation, keeping minimal local state. This provides strong consistency (the server is always the single source of truth) at the cost of heavier network load.

3. **Q: How does AFS's design differ from NFS's, and what trade-off does it make instead?**
   A: AFS caches an entire file locally on the client when opened, working from that local copy until the file is closed, only communicating with the server at open and close. This dramatically reduces network load for repeated access, but weakens consistency — changes aren't visible to other clients' already-open copies until the modifying client closes the file.

## Summary

- A networked file system extends the file abstraction's transparency across a network, letting clients access remote files using the same API as local ones.
- NFS keeps clients minimally stateful, relying on the server for nearly every operation — strong consistency, heavier network load.
- AFS caches entire files locally on the client between open and close — much lighter network load, but weaker consistency across simultaneously-open clients.
- This trade-off (network load vs. consistency) is a recurring, general theme in distributed systems design, not specific to file systems alone — this closes out the module and Persistence as a whole (Modules 17–23), before Module 24 consolidates the entire course's interview preparation.
