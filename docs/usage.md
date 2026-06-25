# Using await-reply

## When to arm

Arm `await-reply` only when **all** of these hold:

- You have sent a message to a participant and expect a reply.
- That reply **blocks your next step.**
- It will not arrive within the current turn.

**Do not** arm when the reply is not blocking (just continue), or when independent
work can proceed first (do that work; arm later if still needed).

## How to arm

Run it **in the background**, pointed at your own inbox, naming the sender whose
reply you await:

```
await-reply <sender> --inbox <your-inbox> [--interval 60] [--timeout 3600]
```

- `<sender>` — the participant whose reply you expect; matches `*_from-<sender>_*.md`.
- `--inbox` — your inbox directory (or set `INBOX_DIR`).
- `--interval` — poll cadence (default 60s).
- `--timeout` — hard cap (default 3600s = 60 min). Raise only for known-slow replies.

**On Claude Code:** launch the command via the Bash tool with
`run_in_background: true`. The process detaches and sleeps; when it exits, the
agent is re-invoked once.

**On another harness:** attach the same command to that harness's background-job +
wake mechanism (see [`claude-code-and-portability.md`](claude-code-and-portability.md) §5).

## On re-wake — act by exit code

| Exit | Meaning | Do |
|---|---|---|
| `0` | Reply landed (path(s) on stdout) | Read it, file it as handled (move to `processed/`), continue. |
| `3` | Timeout — no new reply within the cap | Re-arm with a longer `--timeout`, **or** escalate to a human. Your judgment. |
| `2` | Error (missing inbox, bad args) | Fix and re-arm. **Do not** silently proceed. |

## Discipline (non-negotiable)

- **Bounded, never unbounded.** Always a finite `--timeout`. A wait is a *wait*,
  not a hang.
- **Cancel when mooted.** If the awaited work becomes irrelevant, stop the
  background process — never leave an orphan poller running.
- **Fail to a human on doubt.** Repeated timeouts, or a reply you cannot act on
  cleanly, → escalate. Do not loop forever; the human is the judgment authority.
- **A reply arriving ≠ your question answered.** Exit `0` means *a new message from
  the sender appeared*, not that it resolved your blocker (v1 matches by sender,
  not content). Read it and judge.
