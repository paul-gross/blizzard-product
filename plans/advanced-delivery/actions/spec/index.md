# Actions specifications

The technical contracts behind `epic:advanced-delivery/actions`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. When a hub node is taken, how its outcome routes, pending and its poll,
bounces, and landing only where a chunk's project links stay as `domain/artifacts/delivery.md` and
`initiative:projects`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md) define them. This spec
replaces what a hub node runs, and changes two things about what follows it: the default poll timeout grows from thirty
minutes to three hours, and an answer a node's graph does not route goes to a person at once.

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, `standards/hub-nodes/run-steps.md` becomes the rule
that a hub node names an action and routes every answer it can give, and `standards/hub-nodes/action-purity.md` is new,
owning `bzh:hub-action-purity`. `standards/hub-nodes/env-contract.md` (`bzh:hub-node-env-contract`) is retired with the
environment it governed, along with every citation of it in `architecture/system-shape/seam-positions.md` and
`configuration.md`. `standards/hub-nodes/outcome-protocol.md` (`bzh:hub-node-outcome-protocol`) trades the last stdout
line for the choice a pass answers, and `step-idempotence.md` (`bzh:hub-node-step-idempotence`) the step's `produces:`
skip for the action's own marker skip. `architecture/crash-correctness/hub.md` names the effect writer's crash points in
place of the step loop's. `standards/hub-nodes.md`, `standards/index.md`, `domain/graphs/nodes.md`,
`domain/graphs/edges.md`, `domain/artifacts/delivery.md`, `workflows/feature-delivery.md`,
`verification/blizzard/gaps.md`, and `verification/blizzard/e2e-scenarios/delivery-policy.md` say a hub node takes an
action. `docs/` drops every mention of `run:` ([carry-over.md](./carry-over.md#documentation)). `blizzard-mock`'s seeded
graph (`graph_seed.py`) names no `run:` today; its `deliver` node gains `action: land` in both its row and its
`definition_yaml`.

| Where                            | Read when                                                                                                                                                         |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [actions.md](./actions.md)       | Building actions — the node's `action:`, the catalogue, what a pass is handed and answers, purity and where a pass runs, routing, crash points, testing hub nodes |
| [carry-over.md](./carry-over.md) | Translating stored `run:` lists, escalating what does not translate, and removing the process runner and its token                                                |
