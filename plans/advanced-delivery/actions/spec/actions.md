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
value or a list of them, as `garden_deliver.py`'s repeatable `--delta` and `--proposals` do; `Node` carries the same,
carried from `NodeDoc` at mint by `authoring.py`. `graph_nodes` (`hub/store/schema.py`) gains `action` (String, null)
and `action_args` (Text, JSON, null) and loses `run`, written and read by `hub/store/internal/graph_store.py`.
`NodeShape.is_hub_command_node` becomes `executor is HUB`: every hub node is driven, since every hub node minted from
here on names an action, and one carried over without one is driven to its escalation
([carry-over.md](./carry-over.md#what-does-not-translate)). The graph wire model, `GraphNodeView` (`wire/graph.py`),
carries no command today; it gains `action` and `action_args`, the OpenAPI document with it, and the web graph view
shows a hub node's action.

`validation.py`'s `_check_run` becomes `_check_action`, which reads the action catalogue
([The catalogue](#the-catalogue)) and never the actions themselves: `action` is legal only on `executor: hub` and
required there; it names an action in the catalogue; `with` carries exactly the arguments that action declares; and the
node authors a choice for every outcome the action can answer other than `pending`, so a graph cannot be minted with an
outcome that has nowhere to go. `bounce_cap`, `poll_interval`, `poll_timeout`, the authored `failure` choice, and every
other rule a hub node is held to are unchanged. `run:` is refused at mint, naming `action:` as its replacement; a stored
`definition_yaml` is translated as [carry-over.md](./carry-over.md#translation) describes.

`DEFAULT_POLL_TIMEOUT` (`hub/delivery/hub_node.py`) becomes three hours, and every packaged graph's hub node sets
`poll_timeout: 10800` in place of `1800`. A node's own `poll_timeout` still governs it; versions already stored keep the
value they were minted with.

## The catalogue

The catalogue is the domain's description of every action, with no implementation behind it. `HubActionCatalogue`
(`hub/domain/actions/catalogue.py`) holds one `ActionSpec` per action:

```python
@dataclass(frozen=True)
class ActionSpec:
    name: str
    args: frozenset[str]          # the keys `with:` must carry
    answers: frozenset[str]       # every choice a pass may answer, `pending` included where it may wait
    lands: bool                   # whether a visit holds the branches its chunk lands on (flows epic)
```

Graph validation reads the catalogue, and the executor reads it to know what a visit must be handed, so neither
`hub/domain/` module imports `hub/delivery/`, which `architecture/clean-architecture.md`'s layer walker forbids. The
implementations live in `hub/delivery/actions/`, one per catalogue entry, keyed by name; `tests/test_hub_actions.py`
asserts the two sets are the same. The catalogue is a value handed to validation and to the executor at composition,
never a module-level global either reads, so a test can hand both a catalogue of its own
([Testing hub nodes](#testing-hub-nodes)).

## What an action is handed

An action implements `IHubAction` (`hub/delivery/actions/base.py`):

```python
class IHubAction(Protocol):
    spec: ActionSpec
    async def take(self, visit: Visit, forge: VisitForge, effects: IEffectSink) -> Pass: ...
```

Its three inputs are all it ever touches:

- **`Visit`** (`hub/domain/actions/visit.py`) is a frozen value holding everything the `run:` environment carried, read
  by the executor before the pass begins: the chunk, its graph and node, the epoch, the node's `action_args`, the
  chunk's artifacts, the markers recorded at this node across every epoch — so a re-visit sees the `ci-rerun/<…>`
  signatures earlier visits wrote — its `GitCommits`, its `RepositoryResolution` (`hub/domain/config/repositories.py`)
  with one `ResolvedRepository` per commit pointer, the work items, the feature title, the chunk's public URL, whether
  the node expects git commits, and the project's id and slug. Every field is immutable — frozen dataclasses, tuples,
  and read-only mappings all the way down.
- **`VisitForge`** is a connection to each resolved repository's forge with that repository's credential already bound,
  built by the executor only for an action whose spec `lands`. In this epic it is one `GitHubHandle` per repository, an
  async `httpx` client in `hub/delivery/internal/github_handle.py` built from that repository's own forge address and
  revealed credential, as `LandRun.at` binds each commit to its own target today; the `forges` epic replaces it with the
  forge role Protocols `ForgeRegistry` hands out. An action never sees a secret, only a handle that already carries one.
- **`IEffectSink`** is how an action records anything. An action never writes: it hands each effect —
  `Marker(name, content)`, `RecordReviewFindings`, `RecordGardenDelivery` — to `await effects.record(effect)`, which
  returns once the executor has made the effect durable, with the result of writing it (`recorded`, `invalid`, or
  `fenced` for the two record effects). An action that merges a pull request therefore hands over the `merged/<repo>`
  marker and waits for it before it touches the next repository, so every external effect is durable before the next one
  happens.

`Pass(choice, log, next_poll_at, escalation)` answers with a choice from the action's `answers`, the lines that today
reach a step's `hub-log.<step>` artifact, and, on `pending`, the earliest moment the action wants to be asked again,
when it knows one — a forge's rate-limit reset, say — which the executor writes on the visit's `hub_node_poll` row and
honors beside the node's `poll_interval`. An action that meets something only a person can settle answers `escalation`,
an `EscalationCause` and what it is about, in place of a choice; the executor escalates the chunk with that cause, as it
does for `UNROUTABLE_OUTCOME`, and routes nothing. Later epics add the causes their actions answer.

## Purity

An action is pure of the hub's data: it reads only what it is handed, and writes only through its sink.
`blizzard-context` gains the rule as `bzh:hub-action-purity` (`standards/hub-nodes/action-purity.md`), whose route for
an action that needs more is to declare it as an input, read by the executor into the `Visit`, never to reach for it
inside the pass. Three gates hold it:

- **An import allowlist.** `tests/test_layering.py` gains `test_hub_actions_import_only_their_allowlist`: every module
  under `hub/delivery/actions/` imports only the standard library, `blizzard.foundation` other than
  `blizzard.foundation.store`, `blizzard.hub.domain.actions`, `blizzard.hub.domain.config.repositories` for
  `ResolvedRepository`, the forge handle Protocols, and its own package. The test reuses the walker the file already has
  (`_imported_modules`), checking against an allowlist rather than `_violations`' forbidden prefixes, so an import
  nobody thought to forbid fails as surely as one somebody did.
- **Frozen inputs.** `tests/test_hub_actions.py` walks `Visit`'s field types recursively and fails on any that is not
  immutable.
- **Nothing smuggled in.** The same test constructs every action in `hub/delivery/actions/` with no arguments, so no
  collaborator can arrive through a constructor.

## Where a pass runs

Actions run on one thread of their own, `ActionLoop` (`hub/delivery/action_loop.py`), with its own event loop, started
and stopped in the hub's lifespan. Because an action touches nothing shared, nothing on that thread needs a lock; the
only thing that crosses between it and the rest of the hub is each pass's effect queue. Keeping actions off the API's
event loop also keeps a pass's heavier work — `base_merge.py` reading a three-hundred-file patch — from slowing a
request or the board's event stream.

`HubNodeExecutor._run_locked` keeps everything before and after the step loop — the poll-timeout check, resolution and
its refusals, routing, bounces — and replaces the loop with one pass. On its own thread, which in this epic is the
caller's — `HubNodeExecutor.run`, `IHubNodeExecutor.run` (`hub/domain/execution/apply.py`), and the `hub-advance` route,
a plain `def` FastAPI runs on a worker thread — it reads the `Visit`, reveals each repository's credential and builds
the `VisitForge` for an action that lands, and submits `take` to the `ActionLoop`. It then drains the pass's effect
queue on that same thread, writing each effect through `HubNodeExecutor.record_marker` or the service its record names —
`services.review_findings.record`, `services.garden_delivery.record` — and releasing the action once each is durable.
The `flows` epic's driver takes this role over and does its writing through `to_thread`.

A pass is bounded by `BZ_HUB_ACTION_PASS_SECONDS`, 300 by default. A pass that overruns is cancelled at its next
`await`, answers `pending`, and records a `hub-action-overran` event (warning) naming the action and the chunk, so the
poll timeout, not the pass bound, decides when waiting has gone on too long, and an operator can see a pass that keeps
reaching its limit. Every call a forge handle makes carries a timeout of its own. An exception escaping a pass answers
`failure` with its traceback in the `hub-log.<action>` artifact, as a crashed script does today; the artifact a step
wrote was named for the step (`hub-log.land-every-repo`), so the packaged `resolve` prompt of
`advanced-development-workflow` reads `hub-log.land` instead.

A repository's secret is revealed only for an action that lands, before the pass, with today's refusal
(`hub-log.repository-secret-unavailable`, routed `failure`) when it cannot be. `record-findings`, `garden-deliver`, and
`pass` reveal nothing and no longer fail on a retired secret.

Each action keeps its own idempotency, as the scripts did: a pass reads the markers recorded at its node and skips what
they say is done. The step-level `produces:` skip goes with the step.

## Routing an answer

A pass answers one of its action's `answers`, and the node authors every one of them but `pending`, so a graph minted
from here on always has somewhere to send it. A stored version minted before this epic may not: a carried-over node
whose graph authors no edge for the choice its pass answered — a version that named `land_default` and authors no
`inherited-failure`, say — records an escalation (`EscalationCause.UNROUTABLE_OUTCOME`) naming the node and the choice,
and an `unroutable-outcome` event (critical), at once. The chunk goes to a person, who restarts or migrates it onto a
graph that routes it; it never waits out a poll or retries the pass.

## The actions

| Action            | Arguments            | Answers                                                         | Lands | From                                                    |
| ----------------- | -------------------- | --------------------------------------------------------------- | ----- | ------------------------------------------------------- |
| `land`            | —                    | `landed`, `conflict`, `failure`, `inherited-failure`, `pending` | yes   | `land_pr_ci.py`, `land_common.py`, `base_merge.py`      |
| `record-findings` | —                    | `recorded`, `invalid`, `failure`                                | no    | `review_deliver.py` → `services.review_findings.record` |
| `garden-deliver`  | `delta`, `proposals` | `recorded`, `invalid`, `failure`                                | no    | `garden_deliver.py` → `services.garden_delivery.record` |
| `pass`            | —                    | `success`                                                       | no    | an empty `run:`                                         |

**`land`** (`hub/delivery/actions/land.py`) is `land_pr_ci` moved into the hub with its stages, outcomes, and markers
unchanged — `merged/<repo>`, `delivery-pr/<repo>/<n>`, `delivery-findings`, `delivery-findings/foreign-head`,
`ci-rerun/<…>` — speaking to each repository through its `GitHubHandle`. A pass makes as many forge calls as today's
script does in one run, which for a chunk across several repositories, each with base merges to prove, is dozens; the
pass bound is sized for that rather than for a handful.

**`record-findings`** and **`garden-deliver`** hand their record effect to the sink and answer from its result:
`recorded`, `invalid` where the route answered it, and `failure` for a fenced write, as the script's non-zero exit did.
The executor runs the service when it writes the effect, so the garden's commit check runs there with today's
`CommitResolver` until the `forges` epic moves it into the action.

**`pass`** does nothing and answers `success`. It is the hub node that only routes, which an empty `run:` was.

`land_default.py`, which no packaged graph names since the default graph became a triage router, is deleted with the
other scripts and `hub/graphs/scripts/`; stored versions that still name it are carried onto `land`
([carry-over.md](./carry-over.md#translation)).

## Crash points

The windows a pass opens are the effect writer's, and each is a named point in `bzh:crash-point-registry`
(`foundation/crash.py`), reached by the executor as it writes:

| Point                                    | Window                                                                                                                          | Takes the place of                               |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `action.after-effect.before-write`       | The action's forge write — a merge, a pull request opened, a check re-run — is done; the effect recording it is not yet durable | `hubnode.after-step.before-marker`               |
| `action.after-write.before-resume`       | An effect is durable; the action has not resumed — between one repository's `merged/<repo>` and the next repository's merge     | `hubnode.after-marker.before-next`               |
| `close.after-enqueue.before-drain`       | A `merged/<repo>` marker and its close intents are durable in one transaction; no drain has run yet                             | itself, moved from the step loop                 |
| `hubnode.after-poll.before-slot-release` | The poll-attempt fact is durable; the tenant's slot is not yet released                                                         | itself, unchanged until `flows` retires the slot |

`architecture/crash-correctness/hub.md` names each, and the kill-9 sweep (`tests/crash/test_kill9_sweep.py`) reaches
each through a real hub process armed with `BLIZZARD_CRASH_POINT` and `BLIZZARD_CRASH_FENCE`, including a two-repository
`land` killed between its repositories that, restarted, lands the second without merging the first twice.

## Testing hub nodes

A graph a test writes can no longer name a command, so it names an action. The catalogue and the implementations are
both handed to validation and to the executor, so an in-process test supplies its own. For the tiers that drive a real
hub process — the end-to-end suites and the crash sweep — two test actions join the shipped catalogue only when
`BLIZZARD_TEST_ACTIONS=1` is set beside the crash fence `BLIZZARD_CRASH_FENCE=1`, read once at import as
`foundation/crash.py` reads its fence, so an installation can never mint them:

- **`test.answer`**, with argument `choice`, answers that choice.
- **`test.sequence`**, with argument `choices`, answers each in turn, one per pass, counting its place by a
  `test-sequence/<n>` marker it records.

A test action's `answers` are the values of its argument, so the mint rule that every answer be routed holds for it too.
The graphs tests author today with `command: "true"`, an inline script, or `land_default` move onto `test.answer`,
`test.sequence`, or `land` against the mock forge, and the fake `IHubCommandRunner` they armed goes with the runner.
