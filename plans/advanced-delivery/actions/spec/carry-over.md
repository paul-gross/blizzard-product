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
`GraphReconciliation._parsed` (`hub/graph_sync.py`), which reads `newest_definition_yaml`, translates a `run:` it
matches into its action, so a `definition_yaml` written before the epic still parses and `graph_sync` compares the
packaged graph against it rather than minting a needless version. Nothing else re-parses a stored document: the graph
wire serves `definition_yaml` as it was stored, the record of what was minted, and a person who authors a new graph from
one meets the mint refusal that names `action:`. A graph authored from here on is never translated; `run:` in it is
refused. The second migration restates the map in its own frozen revision rather than importing `legacy_run.py`.

## The migrations

Two manual migrations (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, each frozen once it reaches
`master` (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), with its colocated migration test
(`bzh:colocated-migration-tests`) and a working `downgrade()` that
`tests/test_store_migrations.py::test_migrate_up_and_down` walks. Both only expand (`bzh:store-expand-contract`): each
adds nullable columns and writes rows, and neither drops, renames, or tightens anything a supported release reads, so
`blizzard:compat` finds nothing in either. Each lands with its columns in `contracts/store/hub.json`, which
`blizzard:store-contract` holds to the schema.

The first, in the epic's first landing, adds `graph_nodes.action` (String, null), `graph_nodes.action_args` (Text, JSON,
null), and `hub_node_poll.next_poll_at` (UtcDateTime, null), where a waiting pass's next-poll hint is kept; existing
rows carry none. It writes no row. Its `downgrade()` first gives every hub row whose `run` is null and whose `action` is
set the step list the inverse map below restores, so a version minted onto an action between the landings still runs
under the older hub, then drops the three columns.

The second, in the epic's third landing, sets `action` and `action_args` for every hub `graph_nodes` row in every tenant
that names no action yet — a version minted onto an action since the first landing keeps its own — from its `run`
through the map, adds `legacy_run` (Text, null), and copies into it verbatim the `run` of each row the map does not
match. It leaves `run` where it is. A row left without an action is reported in the migration's log by its graph's id
and name, the graph's `created_at`, the node's name, and the tenant — graphs carry no version column, so a version is
named by the row that minted it — and the executor's escalation names the command from `legacy_run`
([What does not translate](#what-does-not-translate)). Its `downgrade()` clears `action` and `action_args` on every row
whose `run` is set — the rows it translated — and drops `legacy_run`; a row minted onto an action since keeps it, which
a hub carrying the first landing runs, and the first migration's downgrade gives it the inverse map's `run` on the way
further back. Every row the migration found therefore walks back exactly as it was minted.

From the third landing, `graph_nodes.run` is deprecated rather than dropped, since every supported release reads it and
none yet marks it deprecated. Its `Column` in `hub/store/schema.py` carries `info={"deprecated": "<line>"}`, naming the
line of the release that first carries the landing, `contracts/store/hub.json` says so, and a `deprecated` note joins
the landing's others ([The deprecation, then retirement](#the-deprecation-then-retirement)). Outside the migrations, the
hub stops reading and writing it: `graph_store.py` stores a null `run` for every node minted from here and reads a
node's steps from nowhere, since a translated node is taken by its action and an untranslated one is escalated from
`legacy_run`. The column leaves in the retirement that follows the epic. `legacy_run` stays for good: it is the record
of every command the translation could not carry, and nothing retires it.

The inverse map restores the step each action was in the packaged graphs, for a row minted onto an action, which carries
no `run` of its own:

| `action`          | `run` restored                                                                                           |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| `land`            | one step `land-every-repo`, `python3 -m blizzard.hub.graphs.scripts.land_pr_ci`                          |
| `record-findings` | one step `materialize-review-findings`, `python3 -m blizzard.hub.graphs.scripts.review_deliver`          |
| `garden-deliver`  | one step `materialize-garden-artifacts`, `garden_deliver` with a `--delta` and a `--proposals` per value |
| `pass`            | null                                                                                                     |

A test action has no inverse and is never minted outside the test fences, so a downgrade that meets one raises, naming
the row. An older hub does not refuse a node key it does not know, so it reads a stored `definition_yaml` naming
`action:` as a hub node with an empty `run:`; that differs from the packaged graph, so the next sync of the packaged set
mints it afresh, and nothing stalls on it.

The epic's second landing moves the packaged graphs onto actions, which earns one new version of each that has a hub
node — `basic-development-workflow`, `advanced-development-workflow`, `basic-harness-workflow`, `garden-routine`, and
`ideation`; `default` has none — since their nodes now name actions and the land nodes change their `poll_timeout`, as
do the `advanced-development-workflow` prompts that name the landing log. Nothing syncs the packaged set at boot:
`graph_sync` runs from `blizzard hub graph sync` (`POST /api/admin/graphs/sync`) and mints the set into a tenant when it
is created. So minting those versions is a step of the operator's move onto a release carrying the landing, taken after
the hub has moved and with the release's own graphs — an older hub, reading `action:` as a key it does not know, would
mint a delivery node that does nothing. Chunks already pinned stay on the version they hold; only chunks minted after
the new versions take them. The third landing mints nothing: the translation makes each older stored document parse to
what it already was.

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

The fourth landing removes everything a step was handed or ran through that no contract surface carries:

- `IHubCommandRunner` and `CommandResult` (`hub/delivery/command_runner.py`), `SubprocessHubCommandRunner`
  (`hub/delivery/internal/hub_command_runner.py`), and their wiring in `hub/composition.py`.
- The per-chunk working directory a step ran in: `IHubWorkdir` (`hub/delivery/workdir.py`), `FilesystemHubWorkdir`
  (`hub/delivery/internal/hub_workdir.py`), the `hub_workdir_root` the app passes to composition (`hub/app.py`), and
  their wiring. Nothing writes under `data_dir/hub_workdirs/` from here; the durable path itself is the retirement's
  ([The deprecation, then retirement](#the-deprecation-then-retirement)).
- `HubEnv`, every `ENV_*` constant, `PrintedChoice`, `_run_step`, `_NoopStep`, and `FailedStep`
  (`hub/delivery/hub_node.py`). The `bzh:hub-node-env-contract` variables are no longer set anywhere. The bounce
  envelope, which carried the `FailedStep` in `step`, carries `action` beside it: the name of the action whose pass the
  bounce came from, the choice it answered, and its log artifact's name, and the bounce detail reads
  ``"`{action}` answered {choice}"`` where it read the step's exit code. The envelope's schema, `BounceEnvelope`
  (`wire/bounce.py`), gains `action` as an optional field, and `step` stays as it is declared, required and nullable,
  written `null` from here, so the change to `openapi/hub.openapi.json` only adds. `step` is marked deprecated in the
  same landing ([The deprecation, then retirement](#the-deprecation-then-retirement)), and stays until the retirement
  removes it. The board's `parseBounceEnvelope`
  (`web/projects/fleet/src/lib/chunk/chunk-detail/parse-bounce-envelope.ts`) reads `action` as it reads `step`, and the
  bounce card renders the action and its choice where it renders the step and its exit, so an envelope recorded before
  the upgrade shows its step and a later one its action; neither is rewritten. A 0.2 board, met only after its hub walks
  the store back, shows every key of an envelope generically, `action` among them. The crash points around a step give
  way to the effect writer's ([Crash points](./actions.md#crash-points)).
- The marker token: `MarkerAuthority`'s minting and verification (`hub/delivery/marker_auth.py`), the process-level
  `marker_authority` and `hub_marker_callback_base_url` composition builds it from (`hub/composition.py`, `hub/app.py`),
  `MarkerPrincipal` (`hub/delivery/marker_auth.py`) and its place in the `Principal` union, `MARKER_TOKEN_HEADER`, the
  marker branch of `_machine_credential` (`hub/api/auth_session.py`), and `require_marker_authority`
  (`hub/api/marker_auth.py`), so the three routes it guarded admit only a person holding `CHUNK_CONTROL`, through the
  gate it fell through to. A token was only ever minted into a step's environment, so nothing outside the hub held one
  to present. The `garden-delivery-failure` and `review-findings-failure` markers the scripts wrote are written by their
  actions under the same names, so the prompts that read them are unchanged. The visit's resolved repositories, which
  `hold_repositories` pinned for the marker route, reach the action on the `Visit`, and
  `delivery_repo_landed.repository_id` is stamped from the record id each `LandingRecord` carries.
- `BZ_HUB_LAND_TEST_PAUSE_SECONDS`: the kill-9 sweep reaches the windows it held open through armed crash points
  instead.
- `hub/graphs/scripts/` and its tests, whose cases move onto the actions.

The `BZ_FORGE_*` keys in `hub/domain/config/legacy_keys.py` stay: they are the legacy configuration the hub refuses to
start with and imports through `blizzard hub config import-legacy`, not variables handed to a step.

## The deprecation, then retirement

What only the scripts used sits on surfaces `docs/versioning.md` holds a release to, so it leaves as anything released
does: deprecated in one release, removed in a later one. The epic does the first half and is delivered when its four
landings land; the removal is a cleanup that follows it, outside the epic, and nothing after the epic waits for it.

**Deprecated in the third landing.** Each surface below gets a `deprecated` note under `changes/unreleased/`
(`changes/README.md`) whose `removes-in` is the minor after the release that carries it:

- The three routes `POST /api/chunks/{chunk_id}/hub-markers`, `/garden-delivery`, and `/review-findings-delivery`
  (`hub/api/chunks.py`), marked `deprecated=True` so `openapi/hub.openapi.json` says so (`http-api`). They record what
  they recorded; from the fourth landing they admit only a person holding `CHUNK_CONTROL`
  ([What is removed](#what-is-removed)).
- `blizzard hub record-marker` (`cli`). It only ever worked inside a step's environment, so from the third landing it
  cannot succeed; from the fourth, its `ENV_*` imports gone, it refuses at once, saying a hub step no longer runs
  commands and naming the release that removes it.
- `graph_nodes.run` (`store`), marked deprecated on its `Column` and in `contracts/store/hub.json`, and no longer read
  or written by the hub ([The migrations](#the-migrations)).
- The `data/hub_workdirs` durable path (`config`). Nothing writes it from the fourth landing; its `DurablePath` stays,
  its `holds` saying so, and an installation's existing directory is left in place.
- The step's trace span (`trace`). `contracts/traces/dictionary.json` declares a hub span's attributes, never its name,
  so what it marks deprecated are the step span's two, `blizzard.hub.run_step.name` and
  `blizzard.hub.run_step.exit_code`, each naming the `blizzard.hub.action.*` attributes in their place;
  `docs/deployment/tracing.md` marks the `hub run step` span itself deprecated, naming `hub action pass`. The pass's
  span and its `blizzard.hub.action.name` and `blizzard.hub.action.choice` attributes arrive in the first landing
  ([Where a pass runs](./actions.md#where-a-pass-runs)), so both spans are emitted while graphs name both; from the
  third landing the step span is not emitted, since nothing it describes runs, but its attributes stay declared until
  the retirement.

**Deprecated in the fourth landing.** `BounceEnvelope.step` (`http-api`), with `BounceFailedStep` behind it, marked
deprecated on its field so `openapi/hub.openapi.json` says so, under a `deprecated` note of the same form. It stays
required and nullable, and the hub writes it `null`.

The third landing's `changed` note and its `docs/upgrade.md` section tell an operator what they now meet: a graph naming
`run:` is refused at mint, a custom `run:` command escalates its chunk rather than running, and the default poll timeout
is three hours.

**The retirement**, the cleanup that follows the epic, removes them:

- The three routes, with their wire models (`GardenDeliveryRequest` and `GardenDeliveryResponse`, and the marker and
  review-findings request and response models in `wire/chunk.py`), their paths in `openapi/hub.openapi.json`, and the
  generated web client's types for them. `AmbiguousMarkerRepository` (`hub/delivery/marker_auth.py`) and the 409
  `hub/api/chunks.py` answers for it go with them; the ambiguity is refused when the visit is built instead
  ([A merge that has happened](./actions.md#a-merge-that-has-happened)).
- `blizzard hub record-marker` (`hub/cli/marker.py`, its entry in `hub/cli/__init__.py`, and its place in
  `cli/operator_trace.py`'s `_NEVER_TRACED`), and its entry in `contracts/cli/hub.json`, as `blizzard:cli-contract`
  requires.
- `WORKDIRS_DIRNAME` (`hub/layout.py`), its `DurablePath` in `hub/config_keys.py`, and the `data/hub_workdirs` entry in
  `contracts/config/hub.json`. The hub removes `data_dir/hub_workdirs/` at its first boot after the upgrade.
- The step span's constants (`hub/domain/observability/tracing/platform.py`), its two attributes' dictionary entries,
  and its description in `docs/deployment/tracing.md`, raising `blizzard.trace.schema_version` as every breaking trace
  change does.
- `graph_nodes.run` (`store`), in a migration of its own that drops the column and whose `downgrade()` re-adds it,
  filled from `legacy_run` where that is set and from the inverse map otherwise, so a hub of the deprecating line, which
  never reads it, and one before it, which does, both find a `run` to walk back onto. `contracts/store/hub.json` and the
  `Column` in `hub/store/schema.py` go with it.
- `BounceEnvelope.step` and `BounceFailedStep` (`wire/bounce.py`), their schemas in `openapi/hub.openapi.json`, and the
  board's reading of `step`; an envelope recorded before the fourth landing keeps its `detail`, the line every reader of
  a bounce already shows.

Its `removed` notes each `retires` their deprecation, and its `docs/upgrade.md` section names what replaced each. It is
no landing of the epic. It is a chunk of its own, filed beside the epic but belonging to none, and taken up only once an
operator decides a release may carry the break: after a release carrying the deprecations has shipped, and before the
line their `removes-in` names is cut, since the cut refuses to open a line with an unretired promise. Because it removes
routes, a CLI command, a durable path, a span's attributes, and a column, its commit is marked breaking, `feat!`
(`docs/versioning.md`). Its findings pass only as breaks the window admits, on surfaces every supported release marks
deprecated, so it raises `floor` in `wire/window.toml` to the line whose release first carried the deprecations, as
`docs/versioning.md` asks of the change that removes what a released minor deprecated; from it, a runner of an earlier
line is refused registration. So that `floor` rises once, the retirement is taken up together with the cleanups the
`forges` and `flows` epics leave behind, in the same release.

Nothing in this epic crosses the runner's wire, so its landings leave runners untouched; the retirement reaches them
only through the `floor` it raises.

## Tests

The test modules written against the script process, its token, or its routes are rewritten against the actions or
deleted with what they tested: `test_land_scripts.py`, `test_land_common.py`, `test_garden_deliver_script.py`,
`test_review_deliver_script.py`, `test_hub_command_runner.py`, `test_hub_command_node.py`, `test_marker_auth.py`,
`test_hub_marker_auth.py`, `test_hub_cli_record_marker.py`, `test_pin_hub_delivery.py`, `test_garden_delivery_api.py`,
and `test_garden_delivery_guard_race.py`, with the route rows in `test_route_classification.py` and
`test_route_token_authz.py`, and the fixtures in `tests/support.py` and `tests/fleet_world.py` that arm the fake
`IHubCommandRunner`. The rule beyond the list is the one the list follows: any test that arms the command runner or
mints or presents a marker token moves onto an action or goes in the fourth landing, and any that calls one of the three
routes or `record-marker` holds them behind `CHUNK_CONTROL` alone, and the command's refusal, until the retirement takes
it out with them. The two scenarios in `tests/crash/test_kill9_sweep.py` that held a window open with
`BZ_HUB_LAND_TEST_PAUSE_SECONDS` are rewritten onto the armed crash points, and the sweep's family list and CI subsets
follow the points ([Crash points](./actions.md#crash-points)). `scripts/prose-density-baseline.json` drops its entries
for the deleted scripts and tests.

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
and `BZ_HUB_PROJECT`, `docs/backup.md`, `docs/rollback.md`, and `packaging/docker/README.md` say nothing writes
`data/hub_workdirs/` any longer and drop it in the retirement, `docs/deployment/recovery.md` names the effect writer's
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
  between the two, up and back down, with every row the second migration found walking back as it was minted and the
  version minted onto an action given the inverse map's `run` — its colocated migration test and
  `blizzard:store-contract`.
- Every landing is additive to every supported release: `blizzard:compat` reports no finding on any of the four, with
  `graph_nodes.run` declared deprecated, not dropped, from the third — `blizzard:compat`.
- After the third landing no hub code reads or writes `graph_nodes.run` outside the migrations: a node minted from then
  on stores a null `run`, a translated node is taken by its action whatever its `run` holds, and an untranslated one is
  escalated from `legacy_run` — `blizzard:component-test`.
- A 0.2 hub walked back from the third landing's store runs a chunk pinned to a translated version and one minted onto
  `land` between the landings — `blizzard:rehearsal`.
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
- From the first landing, `hub-advance` re-drives a chunk waiting at a node naming an action, and a node naming `run:`
  still takes poll keys only with a non-empty list — `blizzard:component-test`, `blizzard:unit-test`.
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
- A pass's span carries its action and its choice, and the step span's two attributes stand in the dictionary marked
  deprecated — `blizzard:trace-contract`.
- After the fourth landing the three routes answer a person holding `CHUNK_CONTROL`, are marked deprecated in the
  OpenAPI document, and refuse a request carrying no session; `blizzard hub record-marker` refuses naming its removal —
  `blizzard:component-test`.
- `blizzard-mock`'s seeded graph routes every answer of `land`, and seeds against a schema with and without
  `graph_nodes.action` — `blizzard-mock:unit-test`.
- The escalation sentences, a hub node's action in the graph view, and a bounce envelope carrying `action` and a null
  `step` beside one carrying `step` render — `web:unit-test`.
