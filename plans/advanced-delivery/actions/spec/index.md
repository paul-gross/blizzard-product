# Actions specifications

The technical contracts behind `epic:advanced-delivery/actions`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. When a hub node is taken, how its outcome routes, pending and its poll,
bounces, and landing only where a chunk's project links stay as `domain/artifacts/delivery.md` and
`initiative:projects`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md) define them. This spec
replaces what a hub node runs, and changes three things about what follows it: the default poll timeout grows from
thirty minutes to three hours, an answer a node's graph does not route goes to a person at once, and a merge the forge
has made is recorded even when its chunk moved on while the pass ran.

The initiative's epics land one after another — actions, then `forges`, then `flows`, then `policies` — and this is the
first. It assumes nothing of the others, and each of them assumes its four landings are on `master`; none waits for the
cleanup that later removes what the epic deprecates ([carry-over.md](./carry-over.md#the-deprecation-then-retirement)).

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`:

- `architecture/clean-architecture.md`'s `bzh:domain-package-layers` table gains the `actions` row
  ([The domain package](./actions.md#the-domain-package)), with `_DOMAIN_PACKAGE_LAYERS` in `tests/test_layering.py`.
- `standards/hub-nodes/run-steps.md` becomes `standards/hub-nodes/action-shape.md`, the rule that a hub node names an
  action and routes every answer it can give but a reserved wait, under the id `bzh:hub-node-action-shape` in place of
  `bzh:hub-node-run-shape`; every citation of the old id follows, `standards/worker-nodes/declarations.md`'s Scope line
  among them. `standards/hub-nodes/action-purity.md` is new, owning `bzh:hub-action-purity`.
- `architecture/clean-architecture.md`'s layer table notes that the action catalogue is `graph`'s
  ([The catalogue](./actions.md#the-catalogue)), and `architecture/system-shape.md` stops naming
  `IHubCommandRunner`/`IHubWorkdir` among the hub's external seams.
- `standards/hub-nodes/env-contract.md` (`bzh:hub-node-env-contract`) is retired with the environment it governed, along
  with every citation of it in `architecture/system-shape/seam-positions.md` and `configuration.md`.
- `standards/hub-nodes/outcome-protocol.md` (`bzh:hub-node-outcome-protocol`) trades the last stdout line for the choice
  a pass answers and names the reserved waits; `step-idempotence.md` (`bzh:hub-node-step-idempotence`) trades the step's
  `produces:` skip for the action's own marker skip.
- `domain/execution/fencing.md` names the landing record's admission past the fence
  ([A merge that has happened](./actions.md#a-merge-that-has-happened)).
- `architecture/crash-correctness/hub.md` names the effect writer's crash points in place of the step loop's, and
  `architecture/crash-correctness.md`'s registry rule takes `action.after-effect.before-write` as its example where it
  names `hubnode.after-step.before-marker`, and `action.` among the dedicated families.
- `domain/operations.md` gains `hub-action-missing` and `hub-action-overran`
  ([carry-over.md](./carry-over.md#what-does-not-translate)).
- `standards/hub-nodes.md`, `standards/index.md`, `standards/worker-nodes.md`, `domain/graphs/nodes.md`,
  `domain/graphs/edges.md`, `domain/artifacts.md`, `domain/artifacts/delivery.md`, `workflows/feature-delivery.md`,
  `verification/blizzard/gaps.md`, `verification/blizzard/e2e-scenarios/delivery-policy.md`,
  `verification/blizzard/e2e-scenarios/delivery-loop.md`, `verification/blizzard/e2e-scenarios/garden.md` (whose
  scenario mints the packaged `garden-routine` with its `garden_deliver` command verbatim), and
  `verification/blizzard/commands/test-tiers/crash-sweep.md` say a hub node takes an action, and stop naming the
  scripts.
- `exemplars/index.md` points at an exemplar action in place of `land_clean.py`
  ([carry-over.md](./carry-over.md#documentation)).

In `blizzard`, `docs/` drops every mention of `run:` ([carry-over.md](./carry-over.md#documentation)), and the contracts
follow the code: `contracts/store/hub.json` with each migration, and `contracts/traces/dictionary.json` gaining the
pass's attributes with the step span's marked deprecated beside them — the dictionary declares a hub span's attributes,
never its name. `graph_nodes.run`, the step span's two attributes, `contracts/config/hub.json`'s `data/hub_workdirs`
durable path, `contracts/cli/hub.json`'s `record-marker`, and the three marker routes in `openapi/hub.openapi.json` stay
through the deprecation and leave in the retirement that follows the epic
([carry-over.md](./carry-over.md#the-deprecation-then-retirement)). `blizzard-mock`'s seeded graph names no `run:`
today; its `deliver` node gains `action: land` and the choices `land` answers whenever the hub's schema carries the
column ([carry-over.md](./carry-over.md#the-mocks-seeded-graph)).

## Landing order

A deployment runs a pinned build and moves onto the next only when its operator moves it, and a release can be cut from
`master` between any two landings. So the epic arrives as four landings, each leaving `master` releasable — every graph
a hub cut there may meet runs, and a fleet moved onto it can land the next. Each landing is one chunk or a short run of
them; none is behind a flag, because each is safe on its own, and none is breaking.

1. **Actions beside steps.** The catalogue in `graph`, `Visit` and effects, `IHubAction` and the four actions, the
   `ActionLoop`, the pass span, `action_pass_seconds`, `PollPolicy.due_at`, and the first migration. Mint accepts a hub
   node naming exactly one of `action:` and `run:`; the executor drives one naming an action through a pass and one
   naming `run:` through today's step loop, so every stored and packaged graph runs as before.
   `NodeShape.is_hub_command_node` becomes `executor is HUB` here, so `hub-advance` re-drives a node waiting on an
   action as it does one waiting on a script, and a node naming an action takes its poll keys by `_check_action`'s rule
   ([The node](./actions.md#the-node)). The marker routes and their token still serve the scripts. `land` is proven
   against the mock forge end to end here, before any graph the fleet runs names it.
2. **Packaged graphs onto actions.** The packaged graphs name actions and their land nodes `poll_timeout: 10800`,
   `DEFAULT_POLL_TIMEOUT` becomes three hours, and the tests that author hub nodes move onto `land`, `test.answer`, and
   `test.sequence`. Nothing syncs the packaged set at boot — `graph_sync` runs from `blizzard hub graph sync` and when a
   tenant is created — so minting the changed graphs is a step of the operator's move, taken against a hub that already
   carries the first landing ([carry-over.md](./carry-over.md#the-migrations)). From the move on, the fleet lands its
   own work through `land`. `blizzard-mock`'s seeded-graph change merges any time after the first landing, since it
   reads the schema it seeds.
3. **`run:` retired.** The second migration, which translates every stored `run:` and leaves the column in place,
   deprecated and no longer read or written ([carry-over.md](./carry-over.md#the-migrations)), the translation in
   `graph_sync`, `run:` refused at mint, `HUB_ACTION_MISSING` and `UNROUTABLE_OUTCOME` with their sentences and events,
   the boot pass, `hub:hub-node-has-action`, and `HUB_PENDING_CHOICE` folded into `RESERVED_WAITS`. The step loop is now
   unreachable. The landing carries the epic's change notes, so whichever release first stops running `run:` announces
   it: a `changed` note and its `docs/upgrade.md` section for what an operator now meets, and a `deprecated` note for
   each surface only the scripts used ([carry-over.md](./carry-over.md#the-deprecation-then-retirement)).
4. **The door closed.** The process runner, the working directories' writer, `HubEnv`, the marker token, the scripts,
   the test pause, and the documentation that named them ([carry-over.md](./carry-over.md#what-is-removed)). The three
   marker routes stay, deprecated and open only to a person holding `CHUNK_CONTROL`; `blizzard hub record-marker` stays
   and refuses, naming its removal; the `data/hub_workdirs` path and the step span's attributes stay declared. The one
   contract change is additive: the bounce envelope gains an optional `action` beside its `step`, which it marks
   deprecated.

The epic is delivered when its four landings land, and they are what the `forges`, `flows`, and `policies` epics build
on. Removing what the third landing deprecates is not part of the epic: it is a breaking cleanup of its own, filed
beside the epic and taken up only once an operator decides a release may carry the break
([carry-over.md](./carry-over.md#the-deprecation-then-retirement)).

| Where                            | Read when                                                                                                                                                                                                                                                                              |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [actions.md](./actions.md)       | Building actions — the node's `action:`, the catalogue and its reserved waits, the domain package, what a pass is handed and answers, the landing record, purity and where a pass runs, routing, crash points, testing hub nodes                                                       |
| [carry-over.md](./carry-over.md) | Translating stored `run:` lists, the two migrations and their downgrades, the mock's seeded graph, escalating what does not translate, removing the process runner and its token, deprecating and then retiring what only the scripts used, and the tests, docs, and proof that follow |
