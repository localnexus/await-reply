# await-reply — Specification

Normative spec. Any implementation in any language conforms if it satisfies the
**wait predicate**, the **arm/poll/exit algorithm**, the **exit-code contract**, and
the **host-integration contract** below.

---

## 1. Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `sender` | yes | — | The participant id whose reply is awaited. Matches inbox files `*_from-<sender>_*.md`. |
| `inbox` | yes | `$INBOX_DIR` | The directory to watch (the caller's *own* inbox). |
| `interval` | no | `60` (s) | Poll cadence. Positive integer. |
| `timeout` | no | `3600` (s) | Hard cap. Positive integer. **Never unbounded.** |

Invalid or missing required inputs → exit `2` (fail loud), never a silent default
that could watch the wrong place.

---

## 2. The wait predicate (what counts as "the reply arrived")

A match is **a new top-level message from the sender**, defined precisely as:

1. a file in the **top level** of `inbox` (non-recursive — `processed/`, `sent/`,
   and any other subdirectory are excluded);
2. whose name matches `*_from-<sender>_*.md`;
3. whose modification time is **strictly greater than the arm time** (the instant
   the wait began).

The strict `>` is load-bearing: it excludes messages that already existed when the
wait was armed, so a stale prior reply can never satisfy a new wait.

> **Scope (v1): the predicate is existence, not content.** A match means *a new
> message from the sender appeared* — not that it actually answers the question.
> The re-woken caller must read the message and judge. Content-aware matching is a
> deliberate non-goal (see §6).

---

## 3. Algorithm

```
arm_ts   := now()
deadline := arm_ts + timeout

while now() < deadline:
    matched := [ f for f in inbox/*_from-<sender>_*.md   # top-level only
                 if mtime(f) > arm_ts ]
    if matched is non-empty:
        emit each path in matched on stdout
        exit 0
    sleep(interval)

exit 3   # deadline reached, no match
```

- The loop **self-terminates** at the deadline; the timeout cap is the termination
  guarantee. There is no unbounded wait.
- Diagnostics (armed/timeout notices) go to **stderr**; only matched paths go to
  **stdout**, so a caller can consume stdout as a clean result list.

---

## 4. Exit-code contract

The exit code is the primary signal; the re-woken caller branches on it.

| Code | Meaning | Caller should |
|---|---|---|
| `0` | Reply landed — matching path(s) on stdout | Read and process the message(s), continue. |
| `3` | Timeout — no new reply within the cap | Re-arm with a longer timeout, **or** escalate to a human. Judgment call. |
| `2` | Error / bad args (missing inbox, bad option) | Fix and re-arm. **Do not** silently proceed. |

---

## 5. Host-integration contract (the one platform dependency)

`await-reply` is a short-lived foreground process that **sleeps, then exits.** It
provides no wake of its own. To be useful it must be hosted by a platform that:

> **runs the process in the background and re-invokes (or notifies) the caller when
> that process exits.**

That single capability is the entire platform dependency. Everything else in this
spec is platform-neutral.

- **Claude Code:** native. Launch via the Bash tool with `run_in_background: true`;
  the agent is re-invoked when the background task completes. (Details and the
  reliability notes in [`docs/claude-code-and-portability.md`](docs/claude-code-and-portability.md).)
- **Any other harness:** attach the same process to that harness's background-job +
  completion-callback (or wake) mechanism. The script is unchanged; only the
  *arming* differs.

A platform that cannot re-invoke the caller on background completion can still run
`await-reply` synchronously (the caller blocks in-process until it exits) — it just
loses the "idle at zero cost" benefit.

---

## 6. Deliberate non-goals (v1)

Do not pre-build these; add only when a real need appears.

- **Content-aware matching** — judging whether the reply *answers* the request.
  Requires reading message bodies; a heavier implementation.
- **Multi-sender / any-of-N** — waiting on the first of several senders.
- **Non-inbox triggers** — waiting on a database state change or other signal
  rather than an inbox file.
