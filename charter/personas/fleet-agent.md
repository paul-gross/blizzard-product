# `persona:fleet-agent` — the fleet agent

## Who they are

The worker session itself — the one persona that is not a person. It is listed here because blizzard's design quality is largely determined by how *little* cooperation this persona is trusted to provide. Every guarantee the human personas rely on has to hold even when this one is confused, stalled, or wrong.

## What their work looks like

It wakes up holding a chunk, in an environment prepared for it, and works until it finishes, gets stuck, or dies. It cannot be relied on to report honestly, to terminate promptly, or to notice that it has been replaced. The platform's job is to make none of that matter.

## What they need from blizzard

- **An unambiguous lease it cannot accidentally share.** Whatever it believes, only one worker ever validly holds a piece of work.
- **A way to ask a human a question without dying for it.** Ask-and-exit: the question is handed off, the session ends cleanly, and the answer resumes it.
- **A way to report a verdict that will be believed** — structured, fail-safe, and assumed false when absent rather than trusted when missing.
- **Heartbeating that costs it nothing.** Liveness is emitted as a side effect of doing work — never a discipline the agent must remember.

## What serves them

Atomic leases and epochs in the concurrency model, the tool-use heartbeat and fail-safe verdicts in the supervisor, and ask-and-exit in the ask/answer protocol.
