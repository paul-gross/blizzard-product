# Plan — `epic:board`, local slice

> **Shipped.** This plan is frozen: it records what the board's first slice was conceived to be, and its entry in [delivered.md](../../delivered.md) is the record of what landed. Reach and roles are the [remote slice](../board-remote/index.md), in flight.

An operator who cannot see the fleet cannot trust it. The board is the hub's web front — served by the hub daemon itself, opened in a local browser — and it exists to answer the two questions an operator actually asks: *where is everything?* and *what should run next?* It is a pure client of the hub's public API and its live event stream — no private side channel, no direct store access — and like every hub surface it is structurally unable to touch code: the hub holds pointers and assets, never worktrees or transcripts. From day one it is the same app the remote board will be; going remote later adds reach and roles, never a second app.

## What to build

**Observability, live.** Every view reads existing API routes over the event stream: the fleet view — every chunk with its derived status, current node, and runner; the chunk detail — the full record of node history, judgements, artifacts, questions, and route; the ready queue in its current order; the open questions and decisions awaiting a person; and the merge queue from queued to landed.

**Two controls, both queue-shaping.** The board acts only where acting means deciding what the fleet does next, never executing work:

- **Prioritize** — reorder the ready queue. Ordering is a hub-side property, and this is the surface where the operator sets it.
- **Group** — merge unacquired chunks into one, the combined chunk carrying the union of their PM pointers. Grouping widens what a single agent takes on; it never parallelizes a chunk, and it is a human's judgment call — automated bundling stays parked with `epic:batching`.

Gate decisions resolve from the board as well as the CLI, against the same hub route, from day one.

## Artifacts

The board's founding mockups live in [artifacts/](./artifacts/) — three explorations of what watching a fleet should feel like, from the concept round before the build:

| Mockup | What it explores |
|--------|------------------|
| [mission-control.html](./artifacts/mission-control.html) | The one that won: the dark mission-control aesthetic and the three-column shape — fleet, chunk detail, event log — whose design tokens seeded the shared component library. |
| [flow-board.html](./artifacts/flow-board.html) | A kanban-style alternative: one column per workflow node with chunks flowing across, a merge-queue rail, and a detail drawer. |
| [departures.html](./artifacts/departures.html) | The playful extreme: the fleet as a railway departures board — ready to depart, movements log, announcements — kept for its glanceable-status ideas. |
