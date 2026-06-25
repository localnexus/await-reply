# What await-reply Depends On, What It Adds, and Why Nothing Replaces It

This document draws the **dependency boundary**: the single thing `await-reply`
borrows from its host platform, the things the platform does *not* provide, the
net-new layer this package contributes, and how to port it. Claude Code is used as
the worked example because it has a native background-task primitive; the same
analysis applies to any harness.

---

## 1. The one thing it borrows: a wake on background completion

`await-reply` itself only sleeps and exits. The *useful* behavior — that the caller
resumes the instant the reply lands — comes from one host capability:

> **Run a process in the background, and re-invoke (or notify) the caller when that
> process finishes.**

On **Claude Code** this is native: launch the script via the Bash tool with
`run_in_background: true`, and when the background task exits, the agent is
re-invoked. No timer, no scheduled wake — *task completion is the wake.*

That is the **entire** platform dependency. It is small, and it is pluggable: any
harness with a background-job + completion-callback mechanism can host the script
unchanged.

---

## 2. What Claude Code does NOT provide

Claude Code's native features stop short of "wait for a reply." Each adjacent
feature and why it falls short:

| Native feature | What it does | Why it is **not** a reply-waiter |
|---|---|---|
| **Hooks** (`PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `Notification`, …) | Run a command on a **session-lifecycle** event — something *the agent* does | They are triggered by the agent's own state machine, never by an **external** event such as another agent writing a file. They cannot wait. |
| **`FileChanged` hook** | Runs a command when a **named** file is modified | Watches specific filenames, not "any new file in a directory"; fires on modification, not new-file *arrival*; and it **runs a command — it does not wake an idle agent**. Wrong shape for an inbox. |
| **Background task completion** | Re-invokes the caller when a background process exits | This is the primitive we *use* (§1). On its own it has **no notion of what to wait for** — it cannot express "until a reply from X appears." That condition is what we add. |
| **Timed wake / loop** (`ScheduleWakeup`, `/loop`) | Re-invoke after a delay, or repeat on a timer | **Timed, not event-driven.** Each tick wakes the *whole session* (consuming context), so it is an expensive poll, not a cheap idle wait. |
| **Channels** (webhook/chat push) | Push a message **into an already-running session** | Requires a live, listening session; does not wake an idle/blocked agent waiting on a peer's file. |

**Conclusion:** there is **no native Claude Code mechanism that watches an inbox
directory for a peer's reply and resumes an otherwise-idle agent.** The closest
hook (`FileChanged`) is the wrong granularity and does not wake the agent; the
background primitive lacks a wait condition; the timed/loop options poll the whole
session expensively; channels need a live session. The capability has to be built.

---

## 3. What this package adds (the net-new layer)

On top of the bare "wake on completion" primitive, `await-reply` contributes the
four things that primitive lacks:

1. **A wait predicate.** "A *new* message from sender `S` in the inbox" — sender-
   matched, top-level only, strictly newer than arm time (see [`../SPEC.md`](../SPEC.md)
   §2). The platform has no way to express a wait condition; this is it.
2. **A cheap, bounded sleep.** A detached process that costs nothing while idle —
   not a session-level poll that re-wakes (and re-bills) the whole agent each tick.
3. **A termination guarantee.** A hard timeout cap so a wait can never hang, plus
   distinct exit codes (`0` reply / `3` timeout / `2` error) so the re-woken caller
   knows *why* it woke and what to do.
4. **Determinism.** It resolves on an observable filesystem fact (a file exists with
   a newer mtime), independent of any notification delivery — a signal the caller
   can re-check directly.

```
        ┌─────────────────────────────────────────────┐
        │  await-reply (this package)                  │
        │  • wait predicate (new msg from sender)      │
        │  • cheap bounded sleep                       │
        │  • timeout cap + exit-code contract          │   ← net-new
        ├─────────────────────────────────────────────┤
        │  host primitive: wake on background-complete │   ← borrowed (Claude Code: native)
        └─────────────────────────────────────────────┘
```

---

## 4. The net-new capability = await-reply ✕ inbox

Neither piece is the capability alone:

- The **inbox protocol** is a passive mailbox: durable, asynchronous, fire-and-
  forget. It can deliver a reply but cannot tell you *when* one arrives.
- **await-reply** is a wait condition over a directory, but it is meaningless
  without a message convention to watch.

Composed, they yield something **neither the inbox nor the host platform offers**:

> **an event-driven, near-zero-cost blocking wait for an inter-agent reply** —
> turning asynchronous file mailboxes into a synchronous-feeling request/reply
> channel between autonomous agents, with no human in the loop and no compute
> burned while waiting.

That composite is the contribution. The inbox makes messages durable; `await-reply`
makes *waiting* for one cheap and reliable; the host platform supplies only the
underlying wake.

---

## 5. Porting to another harness

Because the platform dependency is a single, well-defined seam (§1), porting is
mechanical:

1. Keep the script and the spec as-is.
2. Replace only the **arming** step: instead of "Bash tool with
   `run_in_background: true`," attach the process to your harness's background-job
   and completion-callback (or wake) mechanism.
3. If your harness has **no** background-wake at all, run `await-reply`
   synchronously — the caller blocks in-process until it exits. You keep the wait
   predicate, the timeout guarantee, and the exit codes; you lose only the
   "idle at zero cost" property.

The wait predicate, timeout, and exit-code contract never change. Only the wake
does.
