# Actions specifications

The technical contracts behind the actions slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. When a hub node is taken, how its outcome routes,
pending and its poll, bounces, and landing only where a chunk's project links stay as `domain/artifacts/delivery.md` and
`epic:projects`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md) define them. This spec
replaces what a hub node runs; it changes nothing about when it runs or what its outcome does.

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. In `blizzard-context`, `standards/hub-nodes/run-steps.md` becomes the rule
that a hub node names an action, and `standards/hub-nodes/env-contract.md` (`bzh:hub-node-env-contract`) is retired with
the environment it governed, along with every citation of it in `architecture/system-shape/seam-positions.md` and
`configuration.md`. `standards/hub-nodes/outcome-protocol.md` (`bzh:hub-node-outcome-protocol`) trades the last stdout
line for the choice a pass answers, and `step-idempotence.md` (`bzh:hub-node-step-idempotence`) the step's `produces:`
skip for the action's own marker skip. `standards/hub-nodes.md`, `standards/index.md`, `domain/graphs/nodes.md`,
`domain/graphs/edges.md`, `domain/artifacts/delivery.md`, `architecture/crash-correctness/hub.md`,
`workflows/feature-delivery.md`, `verification/blizzard/gaps.md`, and
`verification/blizzard/e2e-scenarios/delivery-policy.md` say a hub node takes an action. `docs/` drops every mention of
`run:`. `blizzard-mock`'s seeded graph (`graph_seed.py`) writes its `graph_nodes` rows directly and names no `run:`
today; its `deliver` node's row gains `action: land`.

| Where                            | Read when                                                                                                        |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| [actions.md](./actions.md)       | Building actions — the node's `action:`, the registry, what a pass is handed and answers, the three actions      |
| [carry-over.md](./carry-over.md) | Translating stored `run:` lists, refusing what does not translate, and removing the process runner and its token |
