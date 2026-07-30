# The exec() System Call

## Learning Objectives

By the end of this section you should be able to:
- Explain what `exec()` does and how it differs fundamentally from `fork()`
- Explain why `exec()`, on success, "never returns" to the code that called it
- Explain what does and does not survive an `exec()` call

## Prerequisites

- Topic 4 (The fork() System Call)

## Motivation

`fork()` alone only ever gives you more copies of the *same* running program. But when you type `ls` into your shell, the shell process itself is not the `ls` program — something has to transform a process from running one program into running a completely different one. That transformation is `exec()`, and understanding it precisely is what makes Topic 7's fork/exec design make sense.

## Problem Statement

After a shell calls `fork()` (Topic 4), it now has a child process — but that child is just a copy of the shell itself, still running the shell's own code. If you typed `ls` at the prompt, you don't want a second copy of the shell to keep running as a "child shell" — you want that child process to actually become the `ls` program, running `ls`'s code, using `ls`'s data, from `ls`'s own starting point.

How does a process discard its current program entirely and start running a completely different one, in its place?

## Concept

### Definition

> `exec()` is a system call that replaces the calling process's current program — its code, its data, its stack, its heap — with a brand-new program loaded from disk, and begins executing that new program from its own starting point. Critically, `exec()` does **not** create a new process; the process ID stays the same. Only the program running *inside* that process changes.

### "exec() Never Returns" (On Success)

This is the detail every explanation of `exec()` emphasizes, and it's worth understanding precisely why: `exec()` doesn't "return a result" the way most system calls do, because if it succeeds, the calling program's own code — including the very next line after the `exec()` call — has been completely overwritten and no longer exists in that process at all. There's nothing left to "return to." Execution simply continues, but now inside the *new* program, starting from the new program's own entry point.

`exec()` only returns (with an error code, in the ordinary function-call sense) if it **fails** — for example, if the requested program file doesn't exist. In that specific case, and only that case, the original program's code is still intact (since replacement never happened), and execution genuinely does continue with the next line after the failed `exec()` call.

### What Survives exec(), and What Doesn't

| Survives exec() | Replaced by exec() |
|---|---|
| Process ID (PID) — it's still the same process | All program code and instructions |
| Open file descriptors (by default) | The program's data, heap, and stack contents |
| Parent-child relationships | The program counter (now points into the new program) |

The fact that open file descriptors survive `exec()` unchanged is not a minor detail — it's the exact mechanism that makes I/O redirection possible, explained fully in Topic 7.

## Internal Working (Preview)

```
 Before exec("/bin/ls", ...):        After exec() succeeds:

 ┌───────────────────────┐          ┌───────────────────────┐
 │  Process (PID 500)      │          │  Process (PID 500)      │
 │  Running: shell code     │  exec  │  Running: ls's code       │
 │  Data: shell's variables │ ────►  │  Data: ls's own variables │
 │  Open files: [stdin,      │        │  Open files: [stdin,      │
 │   stdout, stderr]         │        │   stdout, stderr]  ← kept │
 └───────────────────────┘          └───────────────────────┘
        same PID throughout — only the running program itself changed
```

## Real-World Analogy

Think of `exec()` like an actor who, mid-scene, doesn't leave the stage or get replaced by a different actor (that would be more like creating a new process) — instead, the *same* actor, in the *same* physical spot on the same stage, instantly changes costume and starts performing an entirely different play from its very first line, as if they'd been cast in that new play from the start. The theater seat number and stage position (the "process ID") never change — only which script the person standing there is now performing.

## Why exec() Is Designed to Not Create a New Process

Keeping the same process (and PID) across an `exec()` call — rather than exec implicitly creating yet another new process — is precisely what makes the separation from `fork()` (Topic 7) so powerful: it lets a process set up its own environment (open specific files, redirect its own input/output) *before* transforming into a different program, and have that carefully-prepared environment carry over intact into the new program, because it's still, technically, the exact same process the whole time.

## Advantages of This Design

- **Clean separation of "set up the environment" from "run a specific program"** — a process can fully configure itself (file descriptors, working directory, environment variables) and then hand control to any arbitrary program, which inherits that configuration without needing to know or care how it was set up.
- **No wasted process creation** — transforming an existing process is cheaper than creating an entirely new one just to then immediately load different code into it.

## Disadvantages / Costs

- **Irreversible on success** — once `exec()` succeeds, the calling program's own code is gone from that process; there's no way to "exec back" to the original program without having forked a separate copy beforehand (which is exactly why `fork()` is almost always called first — Topic 7).
- **Easy to misunderstand at first** — the "never returns on success" behavior is unlike almost any other system call beginners encounter, and needs to be learned explicitly rather than inferred from typical function-call intuition.

## Best Practices

- Always check `exec()`'s return value for an error — if you see any code executing after an `exec()` call in your own program, you know for certain that `exec()` failed, since success never returns to that point.
- When you want a new program to inherit specific open files (e.g., for I/O redirection), do that setup **before** calling `exec()`, not after — after `exec()` succeeds, your original code (and its ability to make further setup calls) is gone.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "exec() creates a new process to run the new program." | It does not create any new process — it replaces the code and data running inside the *existing* calling process, which keeps the same PID throughout. |
| "The code right after a successful exec() call still runs." | It never runs — a successful exec() has already overwritten that code with the new program entirely. Code after exec() only runs if exec() itself failed. |
| "exec() and fork() do fundamentally the same thing, just with different names." | fork() creates a brand-new, independent process that's a copy of the caller. exec() doesn't create any process at all — it transforms the calling process's own running program into a different one. |

## Interview Questions

1. **Q: What does exec() do, precisely?**
   A: It replaces the calling process's currently running program — its code, data, heap, and stack — with a new program loaded from disk, and begins executing that new program from its own entry point, all within the same process (same PID).

2. **Q: Why is it said that exec() "never returns" on success?**
   A: Because on success, the calling program's own code — including everything after the exec() call — has been entirely overwritten by the new program; there is nothing of the original program left in that process to return control to.

3. **Q: What OS-managed resource typically survives an exec() call, and why does that matter?**
   A: Open file descriptors survive by default. This matters because it lets a process set up specific file descriptors (e.g., redirecting standard output to a file) before calling exec(), and have the new program inherit that exact setup — the core mechanism behind shell I/O redirection (Topic 7).

## Summary

- exec() replaces a process's running program entirely — code, data, heap, stack — while keeping the same process ID.
- It "never returns" on success because the calling code that would have received a return value no longer exists; it only returns (with an error) on failure.
- Open file descriptors survive exec() by default, which is the key mechanism enabling I/O redirection, covered next.
- The next topic covers wait(), how a parent process waits for and collects the result of a child process — whether that child ran the parent's original code, or transformed via exec() into something else entirely.
