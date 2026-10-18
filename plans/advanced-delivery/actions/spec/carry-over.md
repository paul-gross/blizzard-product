# Carry-over

An installation keeps every stored graph, every pinned chunk, and every chunk mid-delivery, carried onto actions across
the epic's four landings ([Landing order](./index.md#landing-order)), with nothing re-pinned by hand. Chunks pinned to
versions that named the packaged landing script land as they did; a version that named an older script lands through
`land` and waits on checks as `land` does.

## Translation

Every `run:` blizzard has shipped is one step running one packaged script, or no step at all, so each maps to one
action:

| A hub node whose `run:` is                                                                      | Becomes                                              |
| ----------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| one step, `python3 -m blizzard.hub.graphs.scripts.land_pr_ci`                                   | `action: land`                                       |
| one step, `python3 -m blizzard.hub.graphs.scripts.review_deliver`                               | `action: record-findings`                            |
| one step, `python3 -m blizzard.hub.graphs.scripts.garden_deliver` with `--delta`, `--proposals` | `action: garden-deliver`, `with: {delta, proposals}` |
| one step, `python3 -m blizzard.hub.graphs.scripts.land_default`                                 | `action: land`                                       |
| one step, `python3 -m blizzard.hub.graphs.scripts.land_ff`                                      | `action: land`                                       |
| empty, or absent                                                                                | `action: pass`                                       |

A row matches on the step's command alone. A stored step is `{command, name, produces}` (`graph_store.py`), and its
`name` and `produces` are dropped in translation: the name only labelled the step's log artifact, which is now named for
the action, and the action's own marker skip takes the place of the `produces:` skip. The command is split as a shell
would split it and matched on the interpreter (`python3` or `python`), `-m`, and the module path, so a difference of
whitespace or quoting never decides it. `garden_deliver`'s arguments are repeatable flags, so its row is a parse, not a
lookup: each `--delta` and `--proposals` value is gathered into its argument's list, and a command carrying any other
flag does not translate. A node with more than one step does not translate, since no action is two scripts; no shipped
graph has had one.

The `land_default` and `land_ff` rows are scripts packaged graphs once named and no longer do: `land_default` was the
default graph's delivery until it became a triage router, and `land_ff` fast-forwarded the basic lanes until they moved
to `land_pr_ci`. Stored versions minted from them remain in every tenant that ran those graphs, and both carry onto
`land`, the one landing the hub keeps. Neither lands exactly as it did. `land_default` merged a pull request GitHub
called clean without reading its checks; under `land` the chunk waits on them, and a repository with no CI waits until
the poll timeout. `land_ff`'s module left the tree with the basic lanes' move and lives only in history, so a chunk
pinned to a version naming it cannot land today and lands through a pull request once carried over. The first
`basic-development-workflow` version to name it authored only `landed` and `conflict`, and other versions of either kind
may author no `inherited-failure`; when `land` answers a choice such a version does not author, the chunk goes to a
person ([Routing an answer](./actions.md#routing-an-answer)).

The map lives in `LEGACY_RUN_COMMANDS` (`hub/domain/graph/legacy_run.py`) and is used where a stored document is parsed:
`graph_sync`'s `_parsed`, which reads `newest_definition_yaml`, translates a `run:` it matches into its action, so a
`definition_yaml` written before the epic still parses and `graph_sync` compares the packaged graph against it rather
than minting a needless version. Nothing else re-parses a stored document: the graph wire serves `definition_yaml` as it
was stored, the record of what was minted, and a person who authors a new graph from one meets the mint refusal that
names `action:`. A graph authored from here on is never translated; `run:` in it is refused. The second migration
restates the map in its own frozen revision rather than importing `legacy_run.py`.

## The migrations

Two manual migrations (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, each frozen once it reaches
`master` (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), with its colocated migration test
(`bzh:colocated-migration-tests`) and a working `downgrade()` that
`tests/test_store_migrations.py::test_migrate_up_and_down` walks. Neither is one-way, so neither is a breaking change
under `docs/versioning.md`. Each lands with its columns in `contracts/store/hub.json`, which `blizzard:store-contract`
holds to the schema.

The first, in the epic's first landing, adds `graph_nodes.action` (String, null), `graph_nodes.action_args` (Text, JSON,
null), and `hub_node_poll.next_poll_at` (UtcDateTime, null), where a waiting pass's next-poll hint is kept; existing
rows carry none. It writes no row. Its `downgrade()` first gives every hub row whose `run` is null and whose `action` is
set the step list the inverse map below restores, so a version minted onto an action between the two landings still runs
under the older hub, then drops the three columns.

The second, in the epic's third landing, sets `action` and `action_args` for every hub `graph_nodes` row in every tenant
from its `run` through the map, adds `legacy_run`, moves into it verbatim the `run` of each row the map does not match,
and drops `run`. A row left without an action is reported in the migration's log by its graph's id and name, the graph's
`created_at`, the node's name, and the tenant — graphs carry no version column, so a version is named by the row that
minted it — and the executor's escalation names the command from `legacy_run`
([What does not translate](#what-does-not-translate)). Its `downgrade()` re-adds `run`, fills it from `legacy_run` where
that is set and from the inverse map otherwise, and drops `legacy_run`.

The inverse map restores the step each action was in the packaged graphs:

| `action`          | `run` restored                                                                                           |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| `land`            | one step `land-every-repo`, `python3 -m blizzard.hub.graphs.scripts.land_pr_ci`                          |
| `record-findings` | one step `materialize-review-findings`, `python3 -m blizzard.hub.graphs.scripts.review_deliver`          |
| `garden-deliver`  | one step `materialize-garden-artifacts`, `garden_deliver` with a `--delta` and a `--proposals` per value |
| `pass`            | null                                                                                                     |

`land_default` and `land_ff` both restore as `land_pr_ci`, the one landing script the older hub still ships. A test
action has no inverse and is never minted outside the test fences, so a downgrade that meets one raises, naming the row.
A downgraded hub's `graph_sync` finds the newest stored `definition_yaml` naming `action:` unparseable, which it already
treats as changed (`graph_sync._parsed`), and mints the packaged graph afresh; nothing stalls on it.

The epic's second landing moves the packaged graphs onto actions, and `graph_sync` mints one new version of each that
has a hub node — `basic-development-workflow`, `advanced-development-workflow`, `basic-harness-workflow`,
`garden-routine`, and `ideation`; `default` has none — since their nodes now name actions and the land nodes change
their `poll_timeout`, as do the `advanced-development-workflow` prompts that name the landing log. Chunks already pinned
stay on the version they hold; only chunks minted after that landing take the new one. The third landing mints nothing:
the translation makes each older stored document parse to what it already was.

## The mock's seeded graph

`blizzard-mock`'s seeded graph (`graph_seed.py`) writes its `graph_nodes` rows and its `definition_yaml` directly,
without minting, so mint's rules never see it, and the seed's drift guard (`schema_contract.py`) refuses a row naming a
column the reflected schema lacks. Its `deliver` node gains `action: land` in both its row and its `definition_yaml`,
and with it the `conflict` and `inherited-failure` choices `land` can answer beside the `landed` and `failure` it
authors today, routed back to the seeded graph's `build` node as its `failure` is, so a seeded chunk's conflict is
routed rather than escalated as unroutable.

The seed writes `action` only when the reflected `graph_nodes` carries the column, and the empty-`run:` shape it writes
today when it does not. That one condition lets `blizzard-mock`'s change merge on its own, immediately after the hub's
first landing, and seed correctly against a hub on either side of it: before the column exists the deliver node routes
as an empty `run:` does, and after it the node takes `land`. No seeded chunk is ever written with a hub node that names
neither, so the hub's third landing never finds one to escalate.

## What does not translate

A node left with no action never runs, and nothing stalls on it unseen. Because every hub node is driven
([The node](./actions.md#the-node)), a chunk that reaches one, or that `hub-advance` finds standing at one, reaches the
executor, which records an escalation (`EscalationCause.HUB_ACTION_MISSING`) naming the node and the command its
`legacy_run` keeps, and a `hub-action-missing` event (critical). Nothing re-parses the stored `definition_yaml` to find
the command. A chunk already standing at such a node when the hub upgrades, with no runner left to call `hub-advance` —
a detached chunk, say — is escalated the same way by a pass the hub makes at boot, after the migration, over every open
tenant's live chunks. Either way the chunk is `needs_human`, and its operator restarts or migrates it onto a graph that
names an action. The escalation is written as every hub-raised escalation is, at the visit's epoch plus one through
`record_bounce_escalation` ([What an action is handed](./actions.md#what-an-action-is-handed)), whose de-duplication on
`(chunk, epoch)` is what makes it once: whichever path reaches the chunk first records it, and the other finds it
recorded and writes nothing, so a chunk the boot pass escalated is never escalated again by the next `hub-advance`.

`EscalationCause` (`foundation/escalation_causes.py`) gains `HUB_ACTION_MISSING` and `UNROUTABLE_OUTCOME`, the OpenAPI
enum with them, and the board's `CAUSE_SENTENCES` (`web/projects/fleet/src/lib/chunk/chunk-detail/chunk-escalation.ts`)
a sentence for each. `EventLogKind`, `EVENT_LOG_SEVERITY` (`foundation/event_log.py`), `tests/event_log_kind_census.py`
at the repository root, and `blizzard-context:/domain/operations.md` gain `hub-action-missing` and `hub-action-overran`;
the unroutable case keeps the `hub-node-unroutable-outcome` kind it is recorded under today, and
`docs/deployment/observability.md` says it now escalates. `tools/invariants.py` gains `hub:hub-node-has-action`: every
hub node at a version a live chunk is pinned to names an action.

## What is removed

- `IHubCommandRunner` and `CommandResult` (`hub/delivery/command_runner.py`), `SubprocessHubCommandRunner`
  (`hub/delivery/internal/hub_command_runner.py`), and their wiring in `hub/composition.py`.
- The per-chunk working directory a step ran in: `IHubWorkdir` (`hub/delivery/workdir.py`), `FilesystemHubWorkdir`
  (`hub/delivery/internal/hub_workdir.py`), the `hub_workdir_root` the app passes to composition (`hub/app.py`), and
  their wiring, with `WORKDIRS_DIRNAME` (`hub/layout.py`), its `DurablePath` in `hub/config_keys.py`, and the
  `data/hub_workdirs` entry in `contracts/config/hub.json`. The hub removes `data_dir/hub_workdirs/` at its first boot
  after the upgrade. Removing a durable path under the runtime root is breaking under `docs/versioning.md`, which the
  landing that removes it marks.
- `HubEnv`, every `ENV_*` constant, `PrintedChoice`, `_run_step`, `_NoopStep`, and `FailedStep`
  (`hub/delivery/hub_node.py`). The `bzh:hub-node-env-contract` variables are no longer set anywhere.
  `BounceEnvelope.step`, which carried the `FailedStep`, becomes `action`: the name of the action whose pass the bounce
  came from, the choice it answered, and its log artifact's name, and the bounce detail reads
  ``"`{action}` answered {choice}"`` where it read the step's exit code. The board renders an envelope's unknown keys
  generically, so envelopes recorded before the upgrade show `step` among their extras and later ones `action`; the
  difference is accepted rather than rewritten. The crash points around a step give way to the effect writer's
  ([Crash points](./actions.md#crash-points)).
- The step's trace span, `RUN_STEP_SPAN` and its `RUN_STEP_*` attributes
  (`hub/domain/observability/tracing/platform.py`), their entries in `contracts/traces/dictionary.json`
  (`blizzard.hub.run_step.*`), and their description in `docs/deployment/tracing.md`. A pass is traced under the live
  `ACTION_PASS_SPAN` the executor opens where the step span opened
  ([Where a pass runs](./actions.md#where-a-pass-runs)); the span and its `blizzard.hub.action.name` and
  `blizzard.hub.action.choice` attributes join the dictionary and `docs/deployment/tracing.md` in the first landing,
  which adds them while the step span still exists, and removing the step span in the last is the trace contract's
  breaking change.
- The marker token: `MarkerAuthority`'s minting and verification (`hub/delivery/marker_auth.py`), the process-level
  `marker_authority` and `hub_marker_callback_base_url` composition builds it from (`hub/composition.py`, `hub/app.py`),
  `MarkerPrincipal` and its place in the `Principal` union, `MARKER_TOKEN_HEADER`, and the marker branch of
  `_machine_credential` (`hub/api/auth_session.py`), `require_marker_authority` (`hub/api/marker_auth.py`), and the
  three routes it guarded — `POST /api/chunks/{chunk_id}/hub-markers`, `/garden-delivery`, and
  `/review-findings-delivery` (`hub/api/chunks.py`) — with their wire models (`GardenDeliveryRequest` and
  `GardenDeliveryResponse`, and the marker and review-findings request and response models in `wire/chunk.py`), their
  paths in `openapi/hub.openapi.json`, and the generated web client's types for them, and `blizzard hub record-marker`
  (`hub/cli/marker.py`, its entry in `hub/cli/__init__.py`, and its place in `cli/operator_trace.py`'s `_NEVER_TRACED`),
  which existed for a script to call the first. Those routes also admitted a person holding `CHUNK_CONTROL`, and that
  operator path goes with them. The `garden-delivery-failure` and `review-findings-failure` markers the scripts wrote
  are written by their actions under the same names, so the prompts that read them are unchanged. The visit's resolved
  repositories, which `hold_repositories` pinned for the marker route, reach the action on the `Visit`, and
  `delivery_repo_landed.repository_id` is stamped from the record id each `LandingRecord` carries.
  `AmbiguousMarkerRepository` (`hub/delivery/marker_auth.py`) and the 409 `hub/api/chunks.py` answers for it go with the
  route; the ambiguity is refused when the visit is built instead
  ([A merge that has happened](./actions.md#a-merge-that-has-happened)). The command's entry in `contracts/cli/hub.json`
  (`record-marker`) goes with it, as `blizzard:cli-contract` requires.
- `BZ_HUB_LAND_TEST_PAUSE_SECONDS`: the kill-9 sweep reaches the windows it held open through armed crash points
  instead.
- `hub/graphs/scripts/` and its tests, whose cases move onto the actions.

The `BZ_FORGE_*` keys in `hub/domain/config/legacy_keys.py` stay: they are the legacy configuration the hub refuses to
start with and imports through `blizzard hub config import-legacy`, not variables handed to a step.

No runner changes: nothing here crosses the runner's wire. Removing three HTTP routes, a durable path, and a trace span
makes the last landing breaking under `docs/versioning.md`, so its commit carries the `!` mark, and its change note
under `changes/` (`changes/README.md`) and a section of `docs/upgrade.md` tell an operator that a custom `run:` command
now escalates its chunk rather than running, and that `blizzard hub record-marker` and its routes are gone.

## Tests

The test modules written against the script process, its token, or its routes are rewritten against the actions or
deleted with what they tested: `test_land_scripts.py`, `test_land_common.py`, `test_garden_deliver_script.py`,
`test_review_deliver_script.py`, `test_hub_command_runner.py`, `test_hub_command_node.py`, `test_marker_auth.py`,
`test_hub_marker_auth.py`, `test_hub_cli_record_marker.py`, `test_pin_hub_delivery.py`, `test_garden_delivery_api.py`,
and `test_garden_delivery_guard_race.py`, with the route rows in `test_route_classification.py` and
`test_route_token_authz.py`, and the fixtures in `tests/support.py` and `tests/fleet_world.py` that arm the fake
`IHubCommandRunner`. The rule beyond the list is the one the list follows: any test that arms the command runner, mints
or presents a marker token, or calls one of the three removed routes moves onto an action or goes. The two scenarios in
`tests/crash/test_kill9_sweep.py` that held a window open with `BZ_HUB_LAND_TEST_PAUSE_SECONDS` are rewritten onto the
armed crash points, and the sweep's family list and CI subsets follow the points
([Crash points](./actions.md#crash-points)). `scripts/prose-density-baseline.json` drops its entries for the deleted
scripts and tests.

The rest follows from changes above rather than from a removal. The thirteen test modules that call `Validator.of(doc)`
directly pass `TEST_CATALOGUE` ([The catalogue](./actions.md#the-catalogue)). Each migration has its colocated test,
covering a stored row of every translated shape, an untranslatable one kept in `legacy_run`, and the walk back down.
`tests/test_container_image.py::test_image_installs_git` stays only if the image keeps `git`
([Documentation](#documentation)). `blizzard-context`'s `verification/blizzard/gaps.md` drops the gaps the script
process left and names any the actions leave.

## Documentation

`docs/` drops every mention of `run:`, and of what a step was handed even where `run:` is not named:
`docs/deployment/repositories.md` stops describing the `BZ_FORGE_*` variables as the land script's,
`docs/deployment/projects.md` stops saying a hub step's `run:` commands see the chunk's project as `BZ_HUB_PROJECT_ID`
and `BZ_HUB_PROJECT`, `docs/backup.md` and `docs/rollback.md` stop listing hub working directories,
`packaging/docker/README.md` drops its `data/hub_workdirs/` row, `docs/deployment/recovery.md` names the effect writer's
crash points in place of the `hubnode.*` step points, keeping `hubnode.after-poll.before-slot-release` until the `flows`
epic retires the slot, `docs/deployment/observability.md` says an unroutable outcome escalates, and
`docs/deployment/install.md` documents `action_pass_seconds`. `packaging/docker/Dockerfile`'s comment that `git` is
there for delivery `run:` scripts cloning into hub workdirs goes: the scripts never cloned, and the image keeps `git`
only if another of its parts still needs it, which the change confirms, keeping or deleting
`tests/test_container_image.py::test_image_installs_git` to match. Each graph's `graph.yaml` comments that name a script
or its route (`land_pr_ci.py`, "the review-findings-delivery route") and `hub/graphs/garden-routine/README.md` name the
action instead.

`blizzard`'s exemplar `exemplars/python/exemplar/hub/graphs/scripts/land_clean.py`, the reference for a hub node's
`run:` step script, which cites `bzh:hub-node-env-contract`, becomes an exemplar action under
`exemplars/python/exemplar/hub/delivery/actions/`, and `blizzard-context`'s `exemplars/index.md` row points at it as the
reference for writing a hub action.

## Proof

Each claim names the method in `blizzard-context`'s verifiability matrix that proves it. The end-to-end and crash
launchers set `BLIZZARD_TEST_ACTIONS=1` beside `BLIZZARD_CRASH_FENCE=1`, so the test actions exist in the hubs they
start and nowhere else.

- Every shipped graph's hub node, minted from its packaged file and from each stored version's `definition_yaml`, names
  the action its script was, and a node with an empty `run:` names `pass` — `blizzard:unit-test`.
- Each migration carries a store of every translated shape, an untranslatable row, and a version minted onto an action
  between the two, up and back down, with `run` restored by the inverse map — its colocated migration test and
  `blizzard:store-contract`.
- A fixture store captured before the migrations, with chunks pinned to old graph versions and one mid-`pending` on
  `land`, carries over and lands every chunk against the mock forge with the outcomes and markers the scripts produced —
  `blizzard:e2e`.
- `record-findings` and `garden-deliver` record what their scripts recorded, and answer `invalid` where they did,
  including a garden delivery with no run context, with the recorder's explanation in the failure marker —
  `blizzard:component-test`.
- A graph naming `run:`, an unknown action, a `with:` key its action does not declare or missing one it requires, an
  action whose answer it leaves unrouted, a choice named `pending`, or poll keys on an action that cannot wait is
  refused at mint — `blizzard:unit-test`.
- A node left without an action escalates its chunk once, at the next epoch, with the command from `legacy_run` named,
  whether the chunk reaches it, is driven at it by `hub-advance`, or stood at it, detached, when the hub upgraded;
  `hub:hub-node-has-action` reports it — `blizzard:component-test`.
- A carried-over version that does not route the choice its pass answered escalates its chunk at once, naming the
  choice, and records `hub-node-unroutable-outcome` once; an action answering `escalation` escalates once, at the next
  epoch, and routes nothing — `blizzard:component-test`.
- A pass that overruns its bound answers `pending` and records `hub-action-overran`, and a pass stuck in synchronous
  work is let go at the caller's deadline with the slot released — `blizzard:component-test` with a small
  `action_pass_seconds`.
- A waiting visit is next due at the later of its interval and its `next_poll_at`, in both the executor's due check and
  `hub-advance`'s detail — `blizzard:unit-test`.
- A chunk stopped while its `land` pass merges its first of two repositories keeps that repository's `merged/<repo>`
  marker and `delivery_repo_landed` row, and its pass answers without touching the second — `blizzard:component-test`.
- The three purity gates fail on a planted action importing the store, `os`, or an HTTP client, on a planted call to
  `asyncio.create_subprocess_exec`, on a planted allowlisted module that imports `subprocess`, on a fixture `Visit`
  holding a list, and on an action constructed with a collaborator; the layer walker fails a planted
  `hub/domain/actions/` import of `execution` and a planted `hub/domain/graph/` import of `hub/domain/actions/` —
  `blizzard:structural-gate`.
- The kill-9 sweep reaches every point in [Crash points](./actions.md#crash-points), and the two-repository `land`
  killed between its repositories lands each exactly once on restart — `blizzard:crash-sweep`.
- End-to-end and crash tiers drive hub nodes through `test.answer` and `test.sequence`, which a hub without both fences
  refuses at mint as unknown actions — `blizzard:e2e`.
- No hub process starts a child process for a graph: no module under `hub/delivery/` reaches `subprocess`, directly or
  through another blizzard module — `blizzard:structural-gate`.
- A pass's span carries its action and its choice, and the step span is gone from the dictionary —
  `blizzard:trace-contract`.
- `blizzard-mock`'s seeded graph routes every answer of `land`, and seeds against a schema with and without
  `graph_nodes.action` — `blizzard-mock:unit-test`.
- The escalation sentences and a hub node's action in the graph view render — `web:unit-test`.
