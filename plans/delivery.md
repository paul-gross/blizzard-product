# Plan — `epic:delivery`

> **Shipped.** This plan is frozen: it records what delivery was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

Landing finished work on a shared main branch is the one lane where the whole fleet converges, and a lane like that is single-writer by nature. The deliberate move is *where* that single writer lives: delivery is a **hub-executed node** — the first of its kind — because only the hub sees every runner, and only there can deliveries be serialized across the entire fleet rather than per machine. By the time a chunk reaches deliver, its branch artifacts are already pushed to the forge, so the hub lands work it never holds: it merges pointers, not code.

## What to build

- **The deliver node, run by the hub's coordinator.** A singleton, runner-like internal executor that acquires the hub node-step, executes it, and records the exit transition under a lease and epoch it mints itself — the same discipline every worker lives under, applied to the hub's own hands.
- **The merge queue.** Strict FIFO, one chunk at a time across all its repos, no train batching. A submission carrying a stale epoch is rejected *before* it enters the queue — a zombie's work never lands, structurally.
- **Serial, convergent multi-repo landing.** A chunk's repos land one at a time with a per-repo landed fact each; cross-repo atomicity is impossible at the forge, so the guarantee is convergence — a failure re-runs the step as reconciliation, already-landed repos detected and skipped, and delivery completes only when every repo has landed.
- **Merge to main, or a pull request at the gate.** The baseline merges; where the graph configures a human gate, deliver opens a PR instead and resolves when it reaches a terminal state — merged by the human, merged by the hub when the workflow says land, or closed without merging, each recorded distinctly. External merges are detected by hub-side polling with an on-demand check for the impatient.
- **Conflicts route through the graph.** A merge conflict is not an error to a human by default — it routes the chunk back through its own graph (the authored conflict edge where one exists, the entry node otherwise), with partial lands retained, into the same warm environments the runner has been holding — because environments ride the chunk's tenure until the terminal outcome, precisely for this moment.

## Why delivery is a node

Because it makes the gate story uniform: a team that wants sign-off before merge inserts a gate node ahead of deliver, and nothing else changes. Process stays configuration, never doctrine.
