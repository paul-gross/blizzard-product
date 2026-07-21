# Plan — `epic:batching`

> **Parked.** This work is deliberately captured and deliberately unscheduled: the solo path it optimizes must be boringly reliable first, and batching's interplay with the workflow graph is still unresolved. Nothing here files to GitHub until the epic is promoted for real.

Twenty small bugs across eight repos should not cost twenty environment setups and twenty premium-model sessions. A naive fleet does exactly that — one task, one environment, one agent, every time. A shrewd one hands the whole batch to one coordinator that fans subagents across a few environments, and pockets the difference. Batching is that shrewdness: shaping open work into chunks that carry many PM items, without ever betting correctness on the shaping being right.

The unit vocabulary is settled: a batch is one **chunk** carrying many PM pointers — there is no second acquirable object. What follows is the shaping machinery.

## The rollout ladder

Each rung is independently useful, and the fleet never leaves a rung until the next one demonstrably earns it:

1. **Strict one-to-one** — the delivered baseline. Every item is its own chunk. Always correct; this floor never goes away.
2. **The heuristic packer.** A deterministic rule bundles the obviously bundleable — same repo, small-labeled items, capped at roughly five per chunk. Most of the savings, no model in the loop.
3. **The LLM planner, shadow first.** One planning call sees the open items and the fleet's state and proposes the batch plan — groupings, sequencing, model and harness routing. It runs *beside* the heuristic without driving, both plans logged and compared, and is promoted to driving only when it demonstrably wins.

## The guardrails

- **A deterministic validator gates every plan.** Every item exists and is unclaimed, no item appears twice, sizes and environment counts are within caps, repo membership is consistent. A plan that fails validation is discarded, and the fleet falls back to one-to-one — the planner can be wrong in judgment, never in arithmetic.
- **Conflict packing.** Items predicted to touch the same files go into the *same* chunk, sequenced — one agent does both, in order. Contention between two agents becomes ordering inside one; every conflict packed is a lease never fought over.
- **Per-member verdicts, always.** A batch reports one verdict per member item, never one verdict for the lot. A batch that fails wholesale explodes: its members requeue as solos, flagged never to batch again.
- **Tighter stall detection.** A batch heartbeats per-member progress, so one silently stuck member is caught faster than a solo would be — and a mid-batch question carries its member's id, so the answer routes to the right work.

## The open question this parks on

How a many-member chunk travels a graph built for one: what a node's judgement means when members diverge — some done, some failed — and what the per-member heartbeat and verdict look like at a node's exit. Until that reconciliation is designed, the packer and planner stay parked.
