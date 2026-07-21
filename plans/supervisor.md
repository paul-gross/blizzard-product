# Plan — `epic:supervisor`

> **Shipped.** This plan is frozen: it records what the runner's supervisor was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

The fleet's keeper is not clever, and that is the whole idea. The supervisor is a small deterministic reconciliation loop that holds no state of its own — every fact lives in the [store](./store.md), every status is derived — so it can be killed at any instant and restarted with no special recovery logic. Its startup pass *is* its normal first step: expire whatever went stale while it was down, then continue. At some point in the night the machine reboots, and it must not matter.

## What to build

**The loop.** Each tick, four idempotent steps in order:

- **Reap** — expire leases whose holder is gone: stale heartbeat or dead process, with pid and recorded start time checked together so a reused pid fools no one. A stalled-but-alive worker is caught by the same clock — heartbeats ride tool use, so an agent that stops progressing stops heartbeating. Under the retry cap, a reaped attempt requeues fresh — new session, new lease, new epoch; over it, the chunk escalates. A reap ends the *attempt*, never the chunk's tenure: it keeps its runner and its environments.
- **Pull** — exchange state with the hub, outbound only: drain the buffered facts upward, pick up delivered answers, and adhere to the brakes — the fleet's pause and the runner's own, either of which stops new work.
- **Fill** — keep the machine busy: for each open agent slot, peek the hub's ordered queue, acquire the chunk's environments from the workspace provider (all-or-nothing — no partial holds, so no deadlock), claim the chunk with one complete route, and spawn a headless worker. Losing a claim race to another runner is ordinary: release and move on.
- **Advance** — judge finished workers and move chunks through the graph: elicit the verdict, submit the completion atomically with its artifacts under the lease's epoch, and continue in place to the next node — same environment, no re-queue.

**Workers heartbeat as a side effect of working.** Progress detection asks nothing of the agent: every tool call emits a heartbeat through a hook. An agent genuinely working cannot help but heartbeat; a stalled one stops on its own. The supervisor never trusts an agent to self-report.

**Environments ride the chunk's tenure.** Bindings are durable facts written at claim time, surviving parks, escalations, and delivery waits; acquisition is idempotent per chunk, so recovery re-reads instead of re-creating. Every acquired environment is clean *by contract* — reset-on-acquire, because reset-on-release can be skipped by exactly the crash that matters.

**Crash-equivalence as a tested property.** `kill -9` at every step boundary — mid-fill, mid-advance, mid-tick — recovers to a correct state: no duplicate environment, no double delivery, no lying status. A tested operation, not a hope.
