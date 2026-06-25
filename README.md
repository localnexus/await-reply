# await-reply — A Reply-Waiting Primitive for File-Based Inboxes

`await-reply` turns a passive file inbox into a **synchronous-feeling request/reply
channel**. After one agent sends a message to another and needs the answer before
it can continue, it *arms* `await-reply` in the background. The primitive sleeps at
near-zero cost and signals the moment the reply lands — or when a deadline passes.

It is the companion to the [**file-based inbox protocol**](https://github.com/localnexus/inbox)
(a separate package). The inbox gives you durable, asynchronous mailboxes;
`await-reply` gives you the ability to *block on a specific reply* without
busy-polling, hand-relaying through a human, or burning compute while you wait. Use
the inbox alone for fire-and-forget messaging; reach for `await-reply` when a reply
is blocking.

```
send a message ──► (inbox protocol delivers it)
       │
   arm await-reply in the background ──► sleeps cheaply
       │                                     │
       │                        a NEW reply from <sender> lands
       │                                     │
       ▼                                     ▼
   caller is re-invoked ◄──── await-reply exits (code 0, paths on stdout)
```

## What problem it solves

An agent that has sent a message and needs the reply has three bad options and one
good one:

| Option | Cost |
|---|---|
| Busy-poll the inbox in a loop | Burns compute every tick; never truly idle |
| Ask a human to relay the reply | Defeats autonomy; adds latency |
| End the turn and hope to be re-triggered later | No guarantee; loses the thread |
| **Arm `await-reply` in the background** | **Sleeps at zero cost; wakes exactly on the event** |

## The key idea: borrow a wake, add the waiting

`await-reply` deliberately does **not** implement its own wake mechanism. It relies
on one capability from the host agent platform: *running a task in the background
and re-invoking (or notifying) the caller when that task finishes.* On top of that
bare primitive it adds everything the primitive lacks — **a wait condition, a cheap
sleep, a guaranteed timeout, and meaningful exit codes.**

The split between *what the platform provides* and *what this package adds* — and
why no current platform feature replaces it — is documented in detail in
[`docs/claude-code-and-portability.md`](docs/claude-code-and-portability.md). Read
that to understand the dependency boundary and how to port it to other harnesses.

## Prerequisites

| Requirement | Detail |
|---|---|
| **A file-based inbox** | A directory whose message files follow `YYYY.MM.DD.HHMM_from-<sender>_<topic>.md` (the inbox protocol). |
| **A POSIX shell + `date`, `stat`, `sleep`** | The reference implementation is portable `bash`; it auto-detects BSD/macOS vs GNU/Linux `stat`. Any language can reimplement the [spec](SPEC.md). |
| **A host that wakes the caller on background-task completion** | This is the one *platform* dependency. On Claude Code it is native (a backgrounded Bash tool call re-invokes the agent on exit). On other harnesses, attach the script to that harness's own background/wake mechanism. |

> **Not required:** any daemon, message broker, database, or network service. The
> primitive is a short-lived process that sleeps and checks a directory.

## Quickstart

1. **Send your message** via the inbox protocol (write to the recipient's inbox).
2. **Arm the wait in the background**, pointing it at *your own* inbox and naming
   the participant whose reply you expect:
   ```
   await-reply <sender> --inbox <path-to-your-inbox> [--interval 60] [--timeout 3600]
   ```
   (Or set `INBOX_DIR` instead of `--inbox`.)
3. **The process sleeps**, polling at `--interval`, until a *new* message from
   `<sender>` appears, or `--timeout` is reached.
4. **On exit, act by exit code:** `0` → read the reply path(s) on stdout and
   continue; `3` → timed out, re-arm or escalate; `2` → fix the error and re-arm.

How to wire step 2 into a specific harness (and the discipline around it) is in
[`docs/usage.md`](docs/usage.md).

## Contents

| Path | What it is |
|---|---|
| [`SPEC.md`](SPEC.md) | Normative spec: the wait predicate, the arm/poll/exit algorithm, exit codes, and the host-integration contract. |
| [`reference/await-reply`](reference/await-reply) | A portable `bash` reference implementation. |
| [`docs/claude-code-and-portability.md`](docs/claude-code-and-portability.md) | What this depends on the platform for, what it adds, why no native feature replaces it, and how to port it. |
| [`docs/usage.md`](docs/usage.md) | Arming it, reading exit codes, cancelling, and the wait discipline. |

## License

See [`LICENSE`](LICENSE).
