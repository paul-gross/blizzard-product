# Plan — `epic:hub`, separation slice

> **Shipped.** This plan is frozen: it records what the hub's first slice was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

A fleet needs a shared memory and a front door, and neither can live inside any single runner. The hub is both: the daemon that knows what work exists, where every chunk stands, and how a human reaches the fleet. The deliberate choice in this slice is to build that daemon **from day one, colocated** — it shares the runner's machine in the solo setup, but the runner speaks to it only through the same outbound-only protocol a remote hub would use. Where the hub runs is a deployment choice; how it is spoken to never changes. That is what makes going remote later a move, not a rewrite.

## What to build

- **The work orchestrator.** The hub wraps the project-management layer: work enters by explicit, id-addressed ingestion — hand it an issue's reference and it mints a chunk holding a pointer, never a copy. Item contents are read pass-through with the hub's own credentials; PM credentials never reach a runner, and runners never speak to the PM system at all. Queue order is a hub-side property — the ready queue's order is what steers the fleet.
- **The record.** The hub owns the workflow definitions and everything fleet-visible about a chunk: current node, transition history, judgements, questions, artifacts, and the route that says which runner, workspace, and environments hold it. The hub always knows where every chunk is.
- **The protocol.** HTTP plus a live event stream, with runners connecting **outbound only** — nothing ever dials into a dev box. The hub is *eventually reachable* by contract: a runner that loses it keeps working what it holds, buffers its facts, and flushes when the hub returns, with per-runner sequence idempotency so replay is the normal drain, not a special case.
- **The registry and the clients.** Runners register and report; the CLI's fleet verbs and the web app read the same API as everything else.
- **Two structural constraints, held throughout.** The hub never stores code or transcripts — pointers and assets only, so it cannot leak what it does not hold — and its store is embedded and private: clients see the API, never the database.

## Open questions carried at the time

The queue-ordering mechanism (explicit rank, aging, starvation protection) and the remote deployment's auth posture — both deferred to later slices rather than half-designed here.
