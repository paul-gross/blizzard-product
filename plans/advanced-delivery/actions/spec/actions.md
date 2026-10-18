# Actions

## The node

A hub node names an action in place of `run:`:

```yaml
# a development graph's landing
deliver:
  executor: hub
  action: land
  poll_interval: 30
  poll_timeout: 10800
---
# garden-routine's delivery
deliver:
  executor: hub
  action: garden-deliver
  with: {delta: delta, proposals: docket}
```

`NodeDoc.run` and `RunStepDoc` (`hub/domain/graph/model.py`) give way to `NodeDoc.action: str | None` and
`NodeDoc.action_args: Mapping[str, tuple[str, ...]]`, parsed from `action` and `with`, where each argument takes one
value or a list of them, as `garden_deliver.py`'s repeatable `--delta` and `--proposals` do; an argument a node leaves
out reads as the empty tuple. `Node` carries the same, carried from `NodeDoc` at mint by `authoring.py`. `graph_nodes`
(`hub/store/schema.py`) gains `action` (String, null) and `action_args` (Text, JSON, null) in the epic's first
migration, and in its second loses `run` and gains `legacy_run` (Text, null), which keeps verbatim the step list of a
row the translation could not match ([carry-over.md](./carry-over.md#the-migrations)); all three are written and read by
`hub/store/internal/graph_store.py`. The graph wire model, `GraphNodeView` (`wire/graph.py`), carries no command today;
it gains `action` and `action_args`, the OpenAPI document with it, and the web graph view shows a hub node's action.
Until the second migration a hub node names exactly one of `action:` and `run:`, and the executor drives a node that
names `run:` through today's step loop ([Landing order](./index.md#landing-order)).

`NodeShape.is_hub_command_node` becomes `executor is HUB`, where today it also asks for a non-empty `run:`. Every hub
node is driven, since every hub node minted from here on names an action, and one carried over without one is driven to
its escalation ([carry-over.md](./carry-over.md#what-does-not-translate)). Its callers follow: `hub_advance_node`
(`hub/domain/chunk/model.py`) answers for a chunk standing at any hub node, so a stored node whose `run:` was empty,
which `hub-advance` never re-drove, is driven from here on and takes `pass`.

`validation.py`'s `_check_run` becomes `_check_action`, which reads the action catalogue
([The catalogue](#the-catalogue)), a module of the `graph` package itself, and never the actions themselves:

- `action` is legal only on `executor: hub` and required there, and it names an action in the catalogue.
- `with` carries only arguments that action declares, each a string or a list of strings, and every argument the action
  requires; an argument it declares but does not require may be left out.
- The node authors a choice for every answer the action can give that is not a reserved wait, so a graph cannot be
  minted with an outcome that has nowhere to go.
- No node authors a choice named by a reserved wait. `pending` is kept out of graphs today by convention alone; from
  here it is refused at mint, by the same membership test every reserved wait a later epic adds passes through.
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
(`hub/domain/graph/actions.py`) holds one `ActionSpec` per action, beside `RESERVED_WAITS`:

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
later epics add their own to this set, in this file, rather than adding a second rule. `HUB_PENDING_CHOICE`
(`hub/domain/graph/model.py`) is retired with it: its readers in `hub/delivery/hub_node.py` read `"pending"` from
`RESERVED_WAITS`' one home, so no second constant names a reserved wait.

The catalogue lives in `graph` because graph validation reads it, and `bzh:domain-package-layers` lets `graph` import
only `artifact`: a catalogue in any higher package would be an upward import the layer walker refuses, whether at module
level, under `TYPE_CHECKING`, or inside a function. `graph/actions.py` imports nothing outside `graph` and the standard
library. The executor reads the same catalogue to know what a visit must be handed, and neither reaches `hub/delivery/`,
which the walker also forbids. The implementations live in `hub/delivery/actions/`, one per catalogue entry, keyed by
name; `tests/test_hub_actions.py` asserts the two sets are the same. The catalogue is a value handed to validation and
to the executor at composition, never a module-level global either reads, so a test can hand both a catalogue of its own
([Testing hub nodes](#testing-hub-nodes)). `Validator.of(doc)` becomes `Validator.of(doc, catalogue)`, and the test
modules that call it directly pass `tests/support.py`'s `TEST_CATALOGUE`, the shipped catalogue with the two test
actions added.

## The domain package

`hub/domain/actions/` is a concept package of its own, and `bzh:domain-package-layers` gains its row: `actions` at L3,
importing `chunk`, `graph`, `artifact`, `config`, and `projects`, beside `kernel`. `blizzard-context`'s table in
`architecture/clean-architecture.md` and `_DOMAIN_PACKAGE_LAYERS` (`tests/test_layering.py`) gain it together, since the
layer walker fails a module that sits in no declared package. Nothing below L3 imports it — graph validation reads the
catalogue in `graph`, not this package — and `execution` does not need to: the executor reaches it from `hub/delivery/`,
an edge.

The package holds `visit.py` and `effects.py`. Three types `hub/delivery/hub_node.py` defines today move into
`visit.py`, since a `Visit` carries them and the domain may not import the delivery edge: `GitCommits`, whose `rows`
becomes a tuple, `UnconvergedDeliveryError`, and `LandingProject`. `effects.py` holds what crosses between a pass and
the hub: `Pass`, the effect types, `EffectResult`, and the `IEffectSink` Protocol.

## What an action is handed

An action implements `IHubAction` (`hub/delivery/actions/base.py`):

```python
class IHubAction(Protocol):
    spec: ActionSpec
    async def take(self, visit: Visit, forges: VisitForges, effects: IEffectSink) -> Pass: ...
```

Its three inputs are all it ever touches:

- **`Visit`** (`hub/domain/actions/visit.py`) is a frozen value holding everything the `run:` environment carried, read
  by the executor before the pass begins: a `VisitChunk` (the chunk's id, project id, minted-at, and work refs), a
  `VisitNode` (the node's id and name, its action, its `action_args`, and whether it expects git commits), the epoch,
  the chunk's artifacts, the markers recorded at this node across every epoch — so a re-visit sees the `ci-rerun/<…>`
  signatures earlier visits wrote — its `GitCommits`, its `RepositoryResolution` (`hub/domain/config/repositories.py`)
  with one `ResolvedRepository` per commit pointer, the work items, the feature title, the chunk's public URL, and its
  `LandingProject`, the project's id and slug. It carries projections, never the domain's `Chunk`, `Graph`, or `Node`:
  those hold lists (`Chunk.work_refs`, `Graph.nodes` and `edges`, `Node.checks`, `produces`, and `choices`) and would
  make the visit mutable through them, and no action reads the graph. Every field is immutable — frozen dataclasses,
  tuples, frozensets, and `types.MappingProxyType` all the way down.
- **`VisitForges`** is a frozen mapping from each resolved repository's id to a handle on that repository's forge, with
  the repository's credential already bound, built by the executor only for an action whose spec `lands`. In this epic
  the handle is `IGitHubHandle`, a Protocol in `hub/delivery/forge_handle.py` that names the GitHub reads and writes
  `land` makes and imports no HTTP client, implemented by `GitHubHandle` in `hub/delivery/internal/github_handle.py`: an
  async `httpx` client built from that repository's own forge address and revealed credential, as `LandRun.at`
  (`land_common.py`) binds each commit to its own target today. Each handle's client is opened on the `ActionLoop` when
  the pass starts and closed there when it ends ([Where a pass runs](#where-a-pass-runs)), so a client never outlives
  its pass or crosses event loops. The `forges` epic replaces the mapping's values with the role handles `ForgeRegistry`
  hands out and deletes all three. An action never sees a secret, only a handle that already carries one.
- **`IEffectSink`** is how an action records anything. An action never writes: it hands each effect to
  `await effects.record(effect)`, which returns once the executor has made the effect durable, with an
  `EffectResult(status, detail)`: `status` is `recorded`, `invalid`, or `fenced`, and `detail` is the service's or the
  fence's own explanation, empty on `recorded`, which an action copies into the failure markers its prompts read. The
  effects are `Marker(name, content)`, `LandingRecord(repository, repo, revision)` — `repository` the record id the
  visit resolved, `repo` the `owner/repo` the marker is named for — `RecordReviewFindings`, and `RecordGardenDelivery`.
  An action that merges a pull request therefore hands over the `LandingRecord` for that repository and waits for it
  before it touches the next repository, so every external effect is durable before the next one happens.

`Pass(choice, log, next_poll_at, escalation)` answers with a choice from the action's `answers_for`, the lines that
today reach a step's `hub-log.<step>` artifact, and, on a reserved wait, the earliest moment the action wants to be
asked again, when it knows one — a forge's rate-limit reset, say — which the executor writes on the visit's
`hub_node_poll` row, in a new nullable `next_poll_at` column ([carry-over.md](./carry-over.md#the-migrations)). The hint
only ever postpones: a waiting visit is next due at the later of its newest row's `polled_at` plus the node's
`poll_interval` and that row's `next_poll_at`. One function, `PollPolicy.due_at(row)`, answers it, and every reader of
due-ness use it — the check `HubNodeExecutor.run` makes before it contends for the tenant's slot, the `next poll at`
detail the `hub-advance` route answers (`hub/api/fleet.py`), and the board's `PendingView.next_poll_at`
(`hub/api/chunk_views.py`), which today adds the interval to `polled_at` itself. An action that meets something only a
person can settle answers `escalation`, an `EscalationCause` and what it is about, in place of a choice; the executor
escalates the chunk with that cause, as it does for `UNROUTABLE_OUTCOME`, and routes nothing. Later epics add the causes
their actions answer.

Every escalation a hub node raises — an action's `escalation`, `UNROUTABLE_OUTCOME`, and `HUB_ACTION_MISSING` — is
written as a bounce-cap escalation is today: through `record_bounce_escalation`
(`hub/store/internal/chunk_escalations_store.py`) at the visit's epoch plus one, with `EpochAdmission.ABOVE` and the
hub's own runner id, so the hub lease and the escalation land in one transaction and the escalation opens a fresh epoch.
A requeue therefore always starts a visit of its own. Each is recorded once per `(chunk, node, epoch)`: the store
de-duplicates on `(chunk, epoch)`, and a visit stands at one node, so a second path to the same escalation — the boot
pass and a `hub-advance`, say — finds it recorded and writes nothing.

## A merge that has happened

A merge on a forge is a fact whether or not the chunk is still where the pass found it. Today `record_marker` writes the
`merged/<repo>` marker through `record_hub_artifact`, whose fence (`fence()`, `hub/store/internal/chunk_rows.py`)
refuses a terminal chunk before it reads the epoch, so a chunk stopped, completed, restarted, or migrated while its land
step merged loses the marker and its `delivery_repo_landed` row though the base already carries the work; the row is
written in a second transaction besides.

`LandingRecord` closes that. The executor writes it in one transaction that holds the `merged/<repo>` marker, its
`delivery_repo_landed` row stamped with the record id the effect carries, and the close intents `enqueue_close_intents`
(`hub/store/internal/chunk_rows.py`) enqueues beside a landed marker in `record_hub_artifact` today, and that
transaction is admitted past the fence: a terminal chunk or a stale epoch does not refuse it, because it records what
the forge already did rather than a step the chunk may take. Replay stays idempotent on `(chunk, node, name, epoch)`.
`blizzard-context:/domain/execution/fencing.md` names the exception beside the stale-attempt rule. When the fence would
have refused the write, the sink answers `fenced` after writing it, and `land` stops there, answering `failure` without
touching another repository, since the chunk it was landing for has moved on. Every other effect is fenced as today, and
answers `fenced` without writing.

Today the marker route resolves a landed marker's `owner/repo` back to a record and refuses one the visit holds for two
records (`AmbiguousMarkerRepository`, `hub/delivery/marker_auth.py`, a 409 from the route). The effect carries the
record id, so the row is never resolved by name; the ambiguity itself, which would leave two landings writing one
`merged/<repo>` marker, is refused earlier, when the executor builds the visit: a resolution that holds one `owner/repo`
for two records answers `failure` before the pass, with `hub-log.repository-ambiguous` naming both records.

## Purity

An action is pure of the hub's data: it reads only what it is handed, and writes only through its sink.
`blizzard-context` gains the rule as `bzh:hub-action-purity` (`standards/hub-nodes/action-purity.md`), whose route for
an action that needs more is to declare it as an input, read by the executor into the `Visit`, never to reach for it
inside the pass. Three gates hold it:

- **An import allowlist.** `tests/test_layering.py` gains `test_hub_actions_import_only_their_allowlist`, which reuses
  the walker the file already has (`_imported_modules`), checking against an allowlist rather than `_violations`'
  forbidden prefixes, so an import nobody thought to forbid fails as surely as one somebody did. Every module under
  `hub/delivery/actions/` imports only what the allowlist names:
  - From the standard library, `__future__`, `abc`, `asyncio`, `base64`, `collections`, `collections.abc`, `contextlib`,
    `dataclasses`, `datetime`, `enum`, `functools`, `hashlib`, `itertools`, `json`, `math`, `re`, `textwrap`, `types`,
    `typing`, and `urllib.parse`. `os`, `sys`, `subprocess`, `socket`, `ssl`, the rest of `urllib`, `http`, `time`
    (whose `sleep` would hold every pass on the loop), `threading`, `multiprocessing`, `importlib`, `shutil`, `pathlib`,
    and `ctypes` are absent by name, so neither `os.environ` — where the key that seals every tenant's secrets lives —
    nor a process nor a socket is reachable by import. Because `asyncio` admits all of itself, the same test also
    refuses any attribute access under `hub/delivery/actions/` to `asyncio.subprocess`,
    `asyncio.create_subprocess_exec`, `asyncio.create_subprocess_shell`, `asyncio.open_connection`,
    `asyncio.open_unix_connection`, and `asyncio.start_server`.
  - From `blizzard.foundation`, only `roles`, `escalation_causes`, `findings`, `garden_proposals`, and `repo_ref`;
    `process`, `store`, `crash`, `logging`, and `trace_export` never join.
  - `blizzard.hub.domain.actions`, `blizzard.hub.domain.graph.actions`, `blizzard.hub.domain.config.repositories` for
    `ResolvedRepository`, `blizzard.hub.delivery.forge_handle` for the handle Protocol, and its own package.

  The check is transitive over blizzard's own modules: every blizzard module an action reaches, directly or through
  another, imports none of `subprocess`, `socket`, `ssl`, `urllib.request`, `http.client`, `multiprocessing`, or
  `ctypes` — the modules that reach a process or the network — and one that imports `os` uses none of it but
  `os.urandom`, so an allowlisted module cannot carry the environment, a process, or a socket in behind it. The
  transitive set is narrower than the direct one on purpose: the domain an action reaches already imports `urllib.parse`
  (`domain/config/repositories.py`) and `threading` (`domain/chunk/dependencies.py`), neither of which reaches past the
  process, and `foundation/ids.py`, which `domain/config/repositories.py` imports for its ids, imports `os` for
  `os.urandom` alone. None of it imports the seven or touches any other part of `os` today. The allowlist is the test's
  own literal, and a later epic that changes it names the module in its spec — the `forges` epic trades
  `blizzard.hub.delivery.forge_handle` for `blizzard.hub.forge.landing`, which holds `RepositoryForge` and
  `VisitForges`, and adds `blizzard.hub.domain.config.capabilities`; the `policies` epic adds
  `blizzard.hub.domain.config.landing_policies`; the `trains` epic widens the walk to `hub/delivery/trains/`. The GitHub
  client sits outside `hub/delivery/actions/` for the same reason: the allowlist admits no HTTP library.
- **Frozen inputs.** `tests/test_hub_actions.py` builds a `Visit` from a fixture visit that exercises every field and
  walks its runtime values recursively, failing on any `list`, `dict`, `set`, `bytearray`, or dataclass instance that is
  not frozen; a mapping passes only as a `types.MappingProxyType` over immutable values. It reads values, never
  annotations, since an annotation of `Mapping` says nothing about the `dict` behind it.
- **Nothing smuggled in.** The same test constructs every action in `hub/delivery/actions/` with no arguments, so no
  collaborator can arrive through a constructor.

## Where a pass runs

Actions run on one thread of their own, `ActionLoop` (`hub/delivery/action_loop.py`), with its own event loop, started
and stopped in the hub's lifespan. Because an action touches nothing shared, nothing on that thread needs a lock; the
only thing that crosses between it and the rest of the hub is each pass's effect queue. Keeping actions off the API's
event loop also keeps a pass's waits on a forge from slowing a request or the board's event stream. The loop is shared
by every pass, so a pass's synchronous work — `base_merge.py` parsing a three-hundred-file patch — holds the others for
as long as it runs; that work is bounded by the patch cap it already refuses past, and an action keeps no other
synchronous work of note. `action_loop.py` imports `threading` to own that thread, which the ast-grep rule
`bzh:store-exclusive-write` flags anywhere under `hub/`; the module carries the rule's `ast-grep-ignore` line, as
`hub/composition.py` does, with the reason beside it: the thread decides nothing exclusive and holds no lock, and every
exactly-one-wins decision a pass leads to is still the store's locked write.

`HubNodeExecutor._run_locked` keeps everything before and after the step loop — the poll-timeout check, resolution and
its refusals, routing, bounces — and replaces the loop with one pass. On its own thread, which in this epic is the
caller's, it reads the `Visit`, reveals each repository's credential and builds the `VisitForges` for an action that
lands, and submits to the `ActionLoop` a wrapper coroutine that opens each handle's client on that loop, awaits `take`,
and closes every client in a `finally`. It then drains the pass's effect queue on that same thread, writing each effect
through `HubNodeExecutor.record_marker`, the landing record's admitted write, or the recorder its record names — the
`ReviewFindingsRecorder` and `GardenDeliveryRecorder` composition builds — and releasing the action once each is
durable. Composition builds both recorders before `HubNodeExecutor` and hands them to it, where today it builds the
executor first (`hub/composition.py`) and the scripts reached the recorders through the routes. The `flows` epic's
driver takes this role over and does its writing through `to_thread`.

The executor opens a live span around each pass on its own thread, `ACTION_PASS_SPAN` (`"hub action pass"`,
`hub/domain/observability/tracing/platform.py`), a child of the visit's linked `HUB_EXEC` context exactly where
`RUN_STEP_SPAN` opens today, carrying `blizzard.hub.action.name` and `blizzard.hub.action.choice`. `HUB_EXEC` itself is
not a live span — it is assembled after the fact from the slot's rows (`tracing/summary.py`, `tracing/assembly.py`) — so
the pass's attributes ride the child that is.

The callers are today's: `HubNodeExecutor.run` through `IHubNodeExecutor.run` (`hub/domain/execution/apply.py`), which
`ApplyService._respond` and `_land_migration` reach inside a worker's completion or a migration, and the `hub-advance`
route, a plain `def` FastAPI runs on a worker thread. Each waits for the pass as it waits for the script now — so until
the `flows` epic, the `ActionLoop` keeps a pass off the API's event loop, but not off the worker thread that asked for
it. A pass may run longer than the runner's hub client waits (`_HTTP_TIMEOUT`, thirty seconds, `runner/loop_wiring.py`);
then, as when a script runs long today, the runner's call times out and is tried again on its next tick, while the pass
runs to its end on the hub holding the tenant's slot, so the retry finds the slot held and defers. Nothing on the runner
changes. The `flows` epic ends the synchronous wait.

A pass is bounded by the hub configuration key `action_pass_seconds`, 300 by default, declared in `hub/config_keys.py`
and `hub/config_defaults.py` beside the hub's other tunables and documented in `docs/deployment/install.md`; like them
it is read from `blizzard-hub.toml`, not the environment, which overrides only `BZ_HUB_HOST`, `BZ_HUB_PORT`, and
`BZ_HUB_DB_URL`. The bound is cooperative: a pass that overruns is cancelled at its next `await`, answers `pending` when
its action can answer a reserved wait and `failure` when it cannot, and records a `hub-action-overran` event (warning)
naming the action and the chunk, so the poll timeout, not the pass bound, decides when waiting has gone on too long, and
an operator can see a pass that keeps reaching its limit. A cancellation that falls between a forge write and its effect
leaves the window the crash points name ([Crash points](#crash-points)), and the next pass reads the forge and converges
as it does after a crash. Every call a forge handle makes carries a timeout of its own.

Cancellation at an `await` is the pass's bound; the caller's is a deadline of its own, so a pass stuck in synchronous
work, or a loop that stops answering, never holds the tenant's slot past it. The caller waits on the pass for
`action_pass_seconds` plus thirty seconds. Reaching that, it closes the sink — a `record` the pass makes afterwards
raises `CancelledError` into it and writes nothing — cancels the task on the `ActionLoop`, and answers as an overrun
does, recording `hub-action-overran`. An effect the caller had already begun writing finishes first, so the deadline
never splits a durable write. An exception escaping a pass answers `failure` with its traceback in the
`hub-log.<action>` artifact, as a crashed script does today; the artifact a step wrote was named for the step
(`hub-log.land-every-repo`), so the packaged `resolve` prompt of `advanced-development-workflow` reads `hub-log.land`
instead.

A repository's secret is revealed only for an action that lands, before the pass, with today's refusal
(`hub-log.repository-secret-unavailable`, routed `failure`) when it cannot be, beside the ambiguity refusal
([A merge that has happened](#a-merge-that-has-happened)). `record-findings`, `garden-deliver`, and `pass` reveal
nothing and no longer fail on a retired secret.

Each action keeps its own idempotency, as the scripts did: a pass reads the markers recorded at its node and skips what
they say is done. The step-level `produces:` skip goes with the step.

## Routing an answer

A pass answers one of its action's answers, and the node authors every one of them that is not a reserved wait, so a
graph minted from here on always has somewhere to send it. A stored version minted before this epic may not. Today
`_route` meets such a choice by recording the `hub-unroutable-outcome` artifact and a `hub-node-unroutable-outcome`
event (critical) once per node and epoch, and then re-polls the same outcome forever. That record stays, under the same
names, and the re-polling ends: the executor also escalates the chunk at once with the new cause
`EscalationCause.UNROUTABLE_OUTCOME`, at the next epoch as every hub-raised escalation is
([What an action is handed](#what-an-action-is-handed)), naming the node and the choice — a version that named
`land_default` and authors no `inherited-failure`, say. The chunk goes to a person, who restarts or migrates it onto a
graph that routes it; it never waits out a poll or retries the pass. A reserved wait is never routed and never
unroutable.

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
chunk" and "unknown node" still answer `invalid`, and the garden's commit check with today's `RepositoryCommitResolver`,
reached through `commit_resolver_for`, until the `forges` epic puts it on the seam. Each answers `invalid` and `failure`
with the recorder's own explanation in its `EffectResult.detail`, which the action writes as the
`garden-delivery-failure` or `review-findings-failure` marker its script wrote. `garden-deliver`'s `delta` and
`proposals` are each a list that may be empty, as the script's flags default to none.

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
`land` killed between its repositories that, restarted, lands the second without merging the first twice. The sweep's
`_DEDICATED_PREFIXES` gains `action.`, with a dedicated scenario that reaches the family through `land` against the mock
forge, and `_ACTION_CI_SUBSET`, `("action.after-effect.before-write",)`, takes the place of `_HUBNODE_CI_SUBSET`; the
`hubnode.` family keeps only its poll point, and `_HUBNODE_PENDING_CI_SUBSET` with it, until `flows`.

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

No shipped action answers `escalation` in this epic, so its path is proven in process: a component test hands the
executor a catalogue holding an action that answers one, and asserts the chunk escalates once, at the next epoch, with
that cause, and that nothing routes.

The graphs tests author today with `command: "true"`, an inline script, or `land_default` move onto `test.answer`,
`test.sequence`, or `land` against the mock forge. The fake `IHubCommandRunner` they armed goes with the runner, and so
does every test written against the script process, the marker token, or its routes;
[carry-over.md](./carry-over.md#tests) names them.
