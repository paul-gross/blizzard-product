# Actions

## The node

A hub node names an action in place of `run:`:

```yaml
deliver:
  executor: hub
  action: land
  poll_interval: 30
  poll_timeout: 10800
deliver:
  executor: hub
  action: garden-deliver
  with: {delta: delta, proposals: docket}
```

`NodeDoc.run` and `RunStepDoc` (`hub/domain/graph/model.py`) give way to `NodeDoc.action: str | None` and
`NodeDoc.action_args: Mapping[str, tuple[str, ...]]`, parsed from `action` and `with`, where each argument takes one
value or a list of them, as `garden_deliver.py`'s repeatable `--delta` and `--proposals` do; an argument a node leaves
out reads as the empty tuple. `Node` carries the same, carried from `NodeDoc` at mint by `authoring.py`. `graph_nodes`
(`hub/store/schema.py`) gains `action` (String, null) and `action_args` (Text, JSON, null) and loses `run`, written and
read by `hub/store/internal/graph_store.py`. The graph wire model, `GraphNodeView` (`wire/graph.py`), carries no command
today; it gains `action` and `action_args`, the OpenAPI document with it, and the web graph view shows a hub node's
action.

`NodeShape.is_hub_command_node` becomes `executor is HUB`, where today it also asks for a non-empty `run:`. Every hub
node is driven, since every hub node minted from here on names an action, and one carried over without one is driven to
its escalation ([carry-over.md](./carry-over.md#what-does-not-translate)). Its callers follow: `hub_advance_node`
(`hub/domain/chunk/model.py`) answers for a chunk standing at any hub node, so a stored node whose `run:` was empty,
which `hub-advance` never re-drove, is driven from here on and takes `pass`.

`validation.py`'s `_check_run` becomes `_check_action`, which reads the action catalogue
([The catalogue](#the-catalogue)) and never the actions themselves:

- `action` is legal only on `executor: hub` and required there, and it names an action in the catalogue.
- `with` carries only arguments that action declares, each a string or a list of strings, and every argument the action
  requires; an argument it declares but does not require may be left out.
- The node authors a choice for every answer the action can give that is not a reserved wait, so a graph cannot be
  minted with an outcome that has nowhere to go.
- No node authors a choice named by a reserved wait. `pending` is kept out of graphs today by review alone; from here it
  is refused at mint.
- `poll_interval` and `poll_timeout` are legal only on a node whose action can answer a reserved wait, so `land` takes
  them and `record-findings`, `garden-deliver`, and `pass` refuse them, as a node without `run:` refuses them today
  (`_check_poll`).

`bounce_cap`, the authored `failure` choice, and every other rule a hub node is held to are unchanged. `run:` is refused
at mint, naming `action:` as its replacement; a stored `definition_yaml` is translated as
[carry-over.md](./carry-over.md#translation) describes.

`DEFAULT_POLL_TIMEOUT` (`hub/delivery/hub_node.py`) becomes three hours, and every packaged graph's hub node sets
`poll_timeout: 10800` in place of `1800`. A node's own `poll_timeout` still governs it; versions already stored keep the
value they were minted with.

## The catalogue

The catalogue is the domain's description of every action, with no implementation behind it. `HubActionCatalogue`
(`hub/domain/actions/catalogue.py`) holds one `ActionSpec` per action, beside `RESERVED_WAITS`:

```python
RESERVED_WAITS: frozenset[str] = frozenset({"pending"})   # answers that are never authored and never a verdict

@dataclass(frozen=True)
class ActionSpec:
    name: str
    args: frozenset[str]              # the keys `with:` may carry
    required_args: frozenset[str]     # the keys it must carry
    answers: frozenset[str]           # every choice a pass may answer, reserved waits included
    answers_from_arg: str | None      # set only by a test action whose answers are its argument's values
    lands: bool                       # whether a visit is handed forge handles and holds its branches

    def answers_for(self, action_args: Mapping[str, tuple[str, ...]]) -> frozenset[str]: ...
```

`answers_for` answers `answers`, or, for an action with `answers_from_arg`, the values its node passed in that argument;
graph validation and routing ask it rather than reading `answers` directly. A reserved wait is an answer that ends a
pass without moving the chunk: never authored, never routed, never a verdict. `pending` is the one this epic reserves;
later epics add their own to the set rather than adding a second rule.

Graph validation reads the catalogue, and the executor reads it to know what a visit must be handed, so neither
`hub/domain/` module imports `hub/delivery/`, which `architecture/clean-architecture.md`'s layer walker forbids. The
implementations live in `hub/delivery/actions/`, one per catalogue entry, keyed by name; `tests/test_hub_actions.py`
asserts the two sets are the same. The catalogue is a value handed to validation and to the executor at composition,
never a module-level global either reads, so a test can hand both a catalogue of its own
([Testing hub nodes](#testing-hub-nodes)).

## The domain package

`hub/domain/actions/` is a concept package of its own, and `bzh:domain-package-layers` gains its row: `actions` at L3,
importing `chunk`, `graph`, `artifact`, `config`, and `projects`, beside `kernel`. `blizzard-context`'s table in
`architecture/clean-architecture.md` and `_DOMAIN_PACKAGE_LAYERS` (`tests/test_layering.py`) gain it together, since the
layer walker fails a module that sits in no declared package. Nothing below L3 imports it, and `execution` does not need
to: the executor reaches it from `hub/delivery/`, an edge.

The package holds `catalogue.py`, `visit.py`, and `effects.py`. Three types `hub/delivery/hub_node.py` defines today
move into `visit.py`, since a `Visit` carries them and the domain may not import the delivery edge: `GitCommits`,
`UnconvergedDeliveryError`, and `LandingProject`. `effects.py` holds what crosses between a pass and the hub: `Pass`,
the effect types, `EffectResult`, and the `IEffectSink` Protocol.

## What an action is handed

An action implements `IHubAction` (`hub/delivery/actions/base.py`):

```python
class IHubAction(Protocol):
    spec: ActionSpec
    async def take(self, visit: Visit, forges: VisitForges, effects: IEffectSink) -> Pass: ...
```

Its three inputs are all it ever touches:

- **`Visit`** (`hub/domain/actions/visit.py`) is a frozen value holding everything the `run:` environment carried, read
  by the executor before the pass begins: the chunk, its graph and node, the epoch, the node's `action_args`, the
  chunk's artifacts, the markers recorded at this node across every epoch — so a re-visit sees the `ci-rerun/<…>`
  signatures earlier visits wrote — its `GitCommits`, its `RepositoryResolution` (`hub/domain/config/repositories.py`)
  with one `ResolvedRepository` per commit pointer, the work items, the feature title, the chunk's public URL, whether
  the node expects git commits, and its `LandingProject`, the project's id and slug. Every field is immutable — frozen
  dataclasses, tuples, and read-only mappings all the way down.
- **`VisitForges`** is a frozen mapping from each resolved repository's id to a handle on that repository's forge, with
  the repository's credential already bound, built by the executor only for an action whose spec `lands`. In this epic
  the handle is `IGitHubHandle`, a Protocol in `hub/delivery/forge_handle.py` that names the GitHub reads and writes
  `land` makes and imports no HTTP client, implemented by `GitHubHandle` in `hub/delivery/internal/github_handle.py`: an
  async `httpx` client built from that repository's own forge address and revealed credential, as `LandRun.at`
  (`land_common.py`) binds each commit to its own target today. The `forges` epic replaces the mapping's values with the
  role handles `ForgeRegistry` hands out and deletes all three. An action never sees a secret, only a handle that
  already carries one.
- **`IEffectSink`** is how an action records anything. An action never writes: it hands each effect to
  `await effects.record(effect)`, which returns once the executor has made the effect durable, with an `EffectResult` of
  `recorded`, `invalid`, or `fenced`. The effects are `Marker(name, content)`, `LandingRecord(repo, revision)`,
  `RecordReviewFindings`, and `RecordGardenDelivery`. An action that merges a pull request therefore hands over the
  `LandingRecord` for that repository and waits for it before it touches the next repository, so every external effect
  is durable before the next one happens.

`Pass(choice, log, next_poll_at, escalation)` answers with a choice from the action's `answers_for`, the lines that
today reach a step's `hub-log.<step>` artifact, and, on a reserved wait, the earliest moment the action wants to be
asked again, when it knows one — a forge's rate-limit reset, say — which the executor writes on the visit's
`hub_node_poll` row, in a new nullable `next_poll_at` column ([carry-over.md](./carry-over.md#the-migration)), and
honors beside the node's `poll_interval`. An action that meets something only a person can settle answers `escalation`,
an `EscalationCause` and what it is about, in place of a choice; the executor escalates the chunk with that cause, as it
does for `UNROUTABLE_OUTCOME`, and routes nothing. Later epics add the causes their actions answer.

## A merge that has happened

A merge on a forge is a fact whether or not the chunk is still where the pass found it. Today `record_marker` writes the
`merged/<repo>` marker through `record_hub_artifact`, whose fence (`fence()`, `hub/store/internal/chunk_rows.py`)
refuses a terminal chunk before it reads the epoch, so a chunk stopped, completed, restarted, or migrated while its land
step merged loses the marker and its `delivery_repo_landed` row though the base already carries the work; the row is
written in a second transaction besides.

`LandingRecord` closes that. The executor writes it in one transaction that holds the `merged/<repo>` marker, its
`delivery_repo_landed` row stamped with the repository id the visit resolved, and the close intents
`enqueue_close_intents` (`hub/store/internal/chunk_rows.py`) enqueues beside a landed marker in `record_hub_artifact`
today, and that transaction is admitted past the fence: a terminal chunk or a stale epoch does not refuse it, because it
records what the forge already did rather than a step the chunk may take. Replay stays idempotent on
`(chunk, node, name, epoch)`. `blizzard-context:/domain/execution/fencing.md` names the exception beside the
stale-attempt rule. When the fence would have refused the write, the sink answers `fenced` after writing it, and `land`
stops there, answering `failure` without touching another repository, since the chunk it was landing for has moved on.
Every other effect is fenced as today, and answers `fenced` without writing.

## Purity

An action is pure of the hub's data: it reads only what it is handed, and writes only through its sink.
`blizzard-context` gains the rule as `bzh:hub-action-purity` (`standards/hub-nodes/action-purity.md`), whose route for
an action that needs more is to declare it as an input, read by the executor into the `Visit`, never to reach for it
inside the pass. Three gates hold it:

- **An import allowlist.** `tests/test_layering.py` gains `test_hub_actions_import_only_their_allowlist`: every module
  under `hub/delivery/actions/` imports only the standard library, `blizzard.foundation` other than
  `blizzard.foundation.store`, `blizzard.hub.domain.actions`, `blizzard.hub.domain.config.repositories` for
  `ResolvedRepository`, `blizzard.hub.delivery.forge_handle` for the handle Protocol, and its own package. The test
  reuses the walker the file already has (`_imported_modules`), checking against an allowlist rather than `_violations`'
  forbidden prefixes, so an import nobody thought to forbid fails as surely as one somebody did. The GitHub client sits
  outside `hub/delivery/actions/` for that reason: the allowlist admits no HTTP library.
- **Frozen inputs.** `tests/test_hub_actions.py` walks `Visit`'s field types recursively and fails on any that is not
  immutable.
- **Nothing smuggled in.** The same test constructs every action in `hub/delivery/actions/` with no arguments, so no
  collaborator can arrive through a constructor.

## Where a pass runs

Actions run on one thread of their own, `ActionLoop` (`hub/delivery/action_loop.py`), with its own event loop, started
and stopped in the hub's lifespan. Because an action touches nothing shared, nothing on that thread needs a lock; the
only thing that crosses between it and the rest of the hub is each pass's effect queue. Keeping actions off the API's
event loop also keeps a pass's waits on a forge from slowing a request or the board's event stream. The loop is shared
by every pass, so a pass's synchronous work — `base_merge.py` parsing a three-hundred-file patch — holds the others for
as long as it runs; that work is bounded by the patch cap it already refuses past, and an action keeps no other
synchronous work of note.

`HubNodeExecutor._run_locked` keeps everything before and after the step loop — the poll-timeout check, resolution and
its refusals, routing, bounces — and replaces the loop with one pass. On its own thread, which in this epic is the
caller's, it reads the `Visit`, reveals each repository's credential and builds the `VisitForges` for an action that
lands, and submits `take` to the `ActionLoop`. It then drains the pass's effect queue on that same thread, writing each
effect through `HubNodeExecutor.record_marker`, the landing record's admitted write, or the service its record names —
`services.review_findings.record`, `services.garden_delivery.record` — and releasing the action once each is durable.
The `flows` epic's driver takes this role over and does its writing through `to_thread`.

The callers are today's: `HubNodeExecutor.run` through `IHubNodeExecutor.run` (`hub/domain/execution/apply.py`), which
`ApplyService._respond` and `_land_migration` reach inside a worker's completion or a migration, and the `hub-advance`
route, a plain `def` FastAPI runs on a worker thread. Each waits for the pass as it waits for the script now. A pass may
run longer than the runner's hub client waits (`_HTTP_TIMEOUT`, thirty seconds, `runner/loop_wiring.py`); then, as when
a script runs long today, the runner's call times out and is tried again on its next tick, while the pass runs to its
end on the hub holding the tenant's slot, so the retry finds the slot held and defers. Nothing on the runner changes.
The `flows` epic ends the synchronous wait.

A pass is bounded by the hub configuration key `action_pass_seconds`, 300 by default, declared in `hub/config_keys.py`
and `hub/config_defaults.py` beside the hub's other tunables and documented in `docs/deployment/install.md`; like them
it is read from `blizzard-hub.toml`, not the environment, which overrides only `BZ_HUB_HOST`, `BZ_HUB_PORT`, and
`BZ_HUB_DB_URL`. The bound is cooperative: a pass that overruns is cancelled at its next `await`, answers `pending` when
its action can answer a reserved wait and `failure` when it cannot, and records a `hub-action-overran` event (warning)
naming the action and the chunk, so the poll timeout, not the pass bound, decides when waiting has gone on too long, and
an operator can see a pass that keeps reaching its limit. A cancellation that falls between a forge write and its effect
leaves the window the crash points name ([Crash points](#crash-points)), and the next pass reads the forge and converges
as it does after a crash. Every call a forge handle makes carries a timeout of its own. An exception escaping a pass
answers `failure` with its traceback in the `hub-log.<action>` artifact, as a crashed script does today; the artifact a
step wrote was named for the step (`hub-log.land-every-repo`), so the packaged `resolve` prompt of
`advanced-development-workflow` reads `hub-log.land` instead.

A repository's secret is revealed only for an action that lands, before the pass, with today's refusal
(`hub-log.repository-secret-unavailable`, routed `failure`) when it cannot be. `record-findings`, `garden-deliver`, and
`pass` reveal nothing and no longer fail on a retired secret.

Each action keeps its own idempotency, as the scripts did: a pass reads the markers recorded at its node and skips what
they say is done. The step-level `produces:` skip goes with the step.

## Routing an answer

A pass answers one of its action's answers, and the node authors every one of them that is not a reserved wait, so a
graph minted from here on always has somewhere to send it. A stored version minted before this epic may not. Today
`_route` meets such a choice by recording the `hub-unroutable-outcome` artifact and a `hub-node-unroutable-outcome`
event (critical) once per node and epoch, and then re-polls the same outcome forever. That record stays, under the same
names, and the re-polling ends: the executor also escalates the chunk at once with the new cause
`EscalationCause.UNROUTABLE_OUTCOME`, naming the node and the choice — a version that named `land_default` and authors
no `inherited-failure`, say. The chunk goes to a person, who restarts or migrates it onto a graph that routes it; it
never waits out a poll or retries the pass. A reserved wait is never routed and never unroutable.

## The actions

| Action            | Arguments                              | Answers                                                         | Lands | From                                                    |
| ----------------- | -------------------------------------- | --------------------------------------------------------------- | ----- | ------------------------------------------------------- |
| `land`            | —                                      | `landed`, `conflict`, `failure`, `inherited-failure`, `pending` | yes   | `land_pr_ci.py`, `land_common.py`, `base_merge.py`      |
| `record-findings` | —                                      | `recorded`, `invalid`, `failure`                                | no    | `review_deliver.py` → `services.review_findings.record` |
| `garden-deliver`  | `delta`, `proposals`, neither required | `recorded`, `invalid`, `failure`                                | no    | `garden_deliver.py` → `services.garden_delivery.record` |
| `pass`            | —                                      | `success`                                                       | no    | an empty `run:`                                         |

**`land`** (`hub/delivery/actions/land.py`) is `land_pr_ci` moved into the hub with its stages, outcomes, and markers
unchanged — `merged/<repo>` (now a `LandingRecord`), `delivery-pr/<repo>/<n>`, `delivery-findings`,
`delivery-findings/foreign-head`, `ci-rerun/<…>` — speaking to each repository through its `IGitHubHandle`. A pass makes
as many forge calls as today's script does in one run, which for a chunk across several repositories, each with base
merges to prove, is dozens; the pass bound is sized for that rather than for a handful.

**`record-findings`** and **`garden-deliver`** hand their record effect to the sink and answer from its result:
`recorded`, `invalid` where the service answered it, and `failure` for a fenced write, as the script's non-zero exit
did. The executor runs the service when it writes the effect, with what the service needs that a pure action cannot
hold: for a garden delivery, `services.run_context.for_chunk`, as the route reads it today, so "no run context for
chunk" and "unknown node" still answer `invalid`, and the garden's commit check with today's `CommitResolver` until the
`forges` epic puts it on the seam. `garden-deliver`'s `delta` and `proposals` are each a list that may be empty, as the
script's flags default to none.

**`pass`** does nothing and answers `success`. It is the hub node that only routes, which an empty `run:` was. A silent
run routed `success` only where the node authored it and `failure` otherwise (`_run_locked`); a node that takes `pass`
authors `success` at mint, and a stored one that does not reaches the person `UNROUTABLE_OUTCOME` sends it to
([carry-over.md](./carry-over.md#translation)).

`land_default.py`, which no packaged graph names since the default graph became a triage router, is deleted with the
other scripts and `hub/graphs/scripts/`; stored versions that still name it are carried onto `land`
([carry-over.md](./carry-over.md#translation)).

## Crash points

The windows a pass opens are the effect writer's, and each is a named point in `bzh:crash-point-registry`
(`foundation/crash.py`), reached by the executor as it writes:

| Point                                    | Window                                                                                                                          | Takes the place of                               |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `action.after-effect.before-write`       | The action's forge write — a merge, a pull request opened, a check re-run — is done; the effect recording it is not yet durable | `hubnode.after-step.before-marker`               |
| `action.after-write.before-resume`       | An effect is durable; the action has not resumed — between one repository's landing record and the next repository's merge      | `hubnode.after-marker.before-next`               |
| `close.after-enqueue.before-drain`       | A landing record and its close intents are durable in one transaction; no drain has run yet                                     | itself, moved from the step loop                 |
| `hubnode.after-poll.before-slot-release` | The poll-attempt fact is durable; the tenant's slot is not yet released                                                         | itself, unchanged until `flows` retires the slot |

`close.after-enqueue.before-drain` is reached today only by a step whose `produces:` names a `merged/` marker, which no
shipped graph declares, so the sweep reaches it through a fixture; under actions every landing record reaches it.
`architecture/crash-correctness/hub.md` names each, and the kill-9 sweep (`tests/crash/test_kill9_sweep.py`) reaches
each through a real hub process armed with `BLIZZARD_CRASH_POINT` and `BLIZZARD_CRASH_FENCE`, including a two-repository
`land` killed between its repositories that, restarted, lands the second without merging the first twice.

## Testing hub nodes

A graph a test writes can no longer name a command, so it names an action. The catalogue and the implementations are
both handed to validation and to the executor, so an in-process test supplies its own. For the tiers that drive a real
hub process — the end-to-end suites and the crash sweep — two test actions join the shipped catalogue only when
`BLIZZARD_TEST_ACTIONS=1` is set beside the crash fence `BLIZZARD_CRASH_FENCE=1`, read once at import as
`foundation/crash.py` reads its fence, so an installation can never mint them:

- **`test.answer`**, with required argument `choice`, answers that choice.
- **`test.sequence`**, with required argument `choices`, answers each in turn, one per pass, counting its place by a
  `test-sequence/<n>` marker it records.

Each sets `answers_from_arg` to its argument, so `answers_for` answers the values its node passed and the mint rule that
every answer be routed holds for it too; a value naming a reserved wait is a wait, not a choice to route.

The graphs tests author today with `command: "true"`, an inline script, or `land_default` move onto `test.answer`,
`test.sequence`, or `land` against the mock forge. The fake `IHubCommandRunner` they armed goes with the runner, and so
does every test written against the script process, the marker token, or its routes;
[carry-over.md](./carry-over.md#tests) names them.
