# Actions specifications

The technical contracts behind `epic:advanced-delivery/actions`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. When a hub node is taken, how its outcome routes, pending and its poll,
bounces, and landing only where a chunk's project links stay as `domain/artifacts/delivery.md` and
`initiative:projects`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md) define them. This spec
replaces what a hub node runs, and changes three things about what follows it: the default poll timeout grows from
thirty minutes to three hours, an answer a node's graph does not route goes to a person at once, and a merge the forge
has made is recorded even when its chunk moved on while the pass ran.

The initiative's epics land one after another — actions, then `forges`, then `flows`, then `policies` — and this is the
first. It assumes nothing of the others, and each of them assumes it has landed.

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`:

- `architecture/clean-architecture.md`'s `bzh:domain-package-layers` table gains the `actions` row
  ([The domain package](./actions.md#the-domain-package)), with `_DOMAIN_PACKAGE_LAYERS` in `tests/test_layering.py`.
- `standards/hub-nodes/run-steps.md` becomes the rule that a hub node names an action and routes every answer it can
  give but a reserved wait, and `standards/hub-nodes/action-purity.md` is new, owning `bzh:hub-action-purity`.
- `standards/hub-nodes/env-contract.md` (`bzh:hub-node-env-contract`) is retired with the environment it governed, along
  with every citation of it in `architecture/system-shape/seam-positions.md` and `configuration.md`.
- `standards/hub-nodes/outcome-protocol.md` (`bzh:hub-node-outcome-protocol`) trades the last stdout line for the choice
  a pass answers and names the reserved waits; `step-idempotence.md` (`bzh:hub-node-step-idempotence`) trades the step's
  `produces:` skip for the action's own marker skip.
- `domain/execution/fencing.md` names the landing record's admission past the fence
  ([A merge that has happened](./actions.md#a-merge-that-has-happened)).
- `architecture/crash-correctness/hub.md` names the effect writer's crash points in place of the step loop's.
- `domain/operations.md` gains `hub-action-missing` and `hub-action-overran`
  ([carry-over.md](./carry-over.md#what-does-not-translate)).
- `standards/hub-nodes.md`, `standards/index.md`, `standards/worker-nodes.md`, `domain/graphs/nodes.md`,
  `domain/graphs/edges.md`, `domain/artifacts.md`, `domain/artifacts/delivery.md`, `workflows/feature-delivery.md`,
  `verification/blizzard/gaps.md`, `verification/blizzard/e2e-scenarios/delivery-policy.md`,
  `verification/blizzard/e2e-scenarios/delivery-loop.md`, and `verification/blizzard/commands/test-tiers/crash-sweep.md`
  say a hub node takes an action, and stop naming the scripts.
- `exemplars/index.md` points at an exemplar action in place of `land_clean.py`
  ([carry-over.md](./carry-over.md#documentation)).

In `blizzard`, `docs/` drops every mention of `run:` ([carry-over.md](./carry-over.md#documentation)). `blizzard-mock`'s
seeded graph names no `run:` today; its `deliver` node gains `action: land` and the choices `land` answers, in a change
that ships with the hub's ([carry-over.md](./carry-over.md#the-mocks-seeded-graph)).

| Where                            | Read when                                                                                                                                                                                                                        |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [actions.md](./actions.md)       | Building actions — the node's `action:`, the catalogue and its reserved waits, the domain package, what a pass is handed and answers, the landing record, purity and where a pass runs, routing, crash points, testing hub nodes |
| [carry-over.md](./carry-over.md) | Translating stored `run:` lists, the mock's seeded graph, escalating what does not translate, removing the process runner and its token, and the tests and docs that follow                                                      |
