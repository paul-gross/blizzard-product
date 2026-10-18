# Carry-over

An installation keeps every stored graph, every pinned chunk, and every chunk mid-delivery, carried onto actions in one
release, with nothing re-pinned by hand. Chunks pinned to versions that named the packaged landing script land as they
did; a version that named an older script lands through `land` and waits on checks as `land` does.

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
names `action:`. A graph authored from here on is never translated; `run:` in it is refused. The migration restates the
map in its own frozen revision rather than importing `legacy_run.py`.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, frozen once released
(`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`). For every hub `graph_nodes` row in every
tenant it sets `action` and `action_args` from `run` through the map, then drops `run`. A row whose `run` the map does
not match keeps no action and is reported by name, graph, version, and tenant in the migration's log; `definition_yaml`
keeps the command for the record. It also adds `hub_node_poll.next_poll_at` (UtcDateTime, null), where a waiting pass's
next-poll hint is kept; existing rows carry none.

`graph_sync` then mints one new version of each packaged graph, since this epic changes their `poll_timeout` and the
`advanced-development-workflow` prompts that name the landing log. Chunks already pinned stay on the version they hold;
only chunks minted after the release take the new one.

## The mock's seeded graph

`blizzard-mock`'s seeded graph (`graph_seed.py`) writes its `graph_nodes` rows and its `definition_yaml` directly,
without minting, so mint's rules never see it and its rows are written against the hub's schema as it stands. Its
`deliver` node gains `action: land` in both its row and its `definition_yaml`, and with it the `conflict` and
`inherited-failure` choices `land` can answer beside the `landed` and `failure` it authors today, routed back to the
seeded graph's `build` node as its `failure` is, so a seeded chunk's conflict is routed rather than escalated as
unroutable. Because the seed writes the schema directly, `blizzard-mock`'s change ships in the same release as the
hub's, and the mock's own tests seed against a hub carrying this epic's migration.

## What does not translate

A node left with no action never runs, and nothing stalls on it unseen. Because every hub node is driven
([The node](./actions.md#the-node)), a chunk that reaches one, or that `hub-advance` finds standing at one, reaches the
executor, which records an escalation (`EscalationCause.HUB_ACTION_MISSING`) naming the node and the command its graph's
`definition_yaml` carried, and a `hub-action-missing` event (critical). A chunk already standing at such a node when the
hub upgrades, with no runner left to call `hub-advance` — a detached chunk, say — is escalated the same way by a pass
the hub makes at boot, after the migration, over every open tenant's live chunks. Either way the chunk is `needs_human`,
and its operator restarts or migrates it onto a graph that names an action. The escalation is recorded once per
`(chunk, node, epoch)`: whichever path reaches the chunk first records it, and the other finds it recorded and writes
nothing, so a chunk the boot pass escalated is never escalated again by the next `hub-advance`.

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
  their wiring. The hub removes `data_dir/hub_workdirs/` at its first boot after the upgrade.
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
  (`blizzard.hub.run_step.*`), and their description in `docs/deployment/tracing.md`; a pass is traced under the visit's
  `HUB_EXEC` span, which gains `blizzard.hub.action.name` and `blizzard.hub.action.choice`, both added to the
  dictionary.
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
  `delivery_repo_landed.repository_id` is stamped from them.
- `BZ_HUB_LAND_TEST_PAUSE_SECONDS`: the kill-9 sweep reaches the windows it held open through armed crash points
  instead.
- `hub/graphs/scripts/` and its tests, whose cases move onto the actions.

The `BZ_FORGE_*` keys in `hub/domain/config/legacy_keys.py` stay: they are the legacy configuration the hub refuses to
start with and imports through `blizzard hub config import-legacy`, not variables handed to a step.

No runner changes: nothing here crosses the runner's wire.

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
armed crash points. `scripts/prose-density-baseline.json` drops its entries for the deleted scripts and tests.

## Documentation

`docs/` drops every mention of `run:`, and of what a step was handed even where `run:` is not named:
`docs/deployment/repositories.md` stops describing the `BZ_FORGE_*` variables as the land script's,
`docs/deployment/projects.md` stops saying a hub step's `run:` commands see the chunk's project as `BZ_HUB_PROJECT_ID`
and `BZ_HUB_PROJECT`, `docs/backup.md` stops listing hub working directories, `docs/deployment/recovery.md` names the
effect writer's crash points in place of the `hubnode.*` step points, keeping `hubnode.after-poll.before-slot-release`
until the `flows` epic retires the slot, `docs/deployment/observability.md` says an unroutable outcome escalates, and
`docs/deployment/install.md` documents `action_pass_seconds`. `packaging/docker/Dockerfile`'s comment that `git` is
there for delivery `run:` scripts cloning into hub workdirs goes: the scripts never cloned, and the image keeps `git`
only if another of its parts still needs it, which the change confirms.

`blizzard`'s exemplar `exemplars/python/exemplar/hub/graphs/scripts/land_clean.py`, the reference for a hub node's
`run:` step script, which cites `bzh:hub-node-env-contract`, becomes an exemplar action under
`exemplars/python/exemplar/hub/delivery/actions/`, and `blizzard-context`'s `exemplars/index.md` row points at it as the
reference for writing a hub action.

## Proof

- Every shipped graph's hub node, minted from its packaged file and from each stored version's `definition_yaml`, names
  the action its script was, and a node with an empty `run:` names `pass`.
- A fixture store captured before the migration, with chunks pinned to old graph versions and one mid-`pending` on
  `land`, carries over and lands every chunk against the mock forge with the outcomes and markers the scripts produced.
- `record-findings` and `garden-deliver` record what their scripts recorded, and answer `invalid` where they did,
  including a garden delivery with no run context.
- A graph naming `run:`, an unknown action, a `with:` key its action does not declare or missing one it requires, an
  action whose answer it leaves unrouted, a choice named `pending`, or poll keys on an action that cannot wait is
  refused at mint.
- A node left without an action escalates its chunk once, with the command named, whether the chunk reaches it, is
  driven at it by `hub-advance`, or stood at it, detached, when the hub upgraded; `hub:hub-node-has-action` reports it.
- A carried-over version that does not route the choice its pass answered escalates its chunk at once, naming the
  choice, and records `hub-node-unroutable-outcome` once.
- A pass that overruns its bound answers `pending` and records `hub-action-overran`.
- A chunk stopped while its `land` pass merges its first of two repositories keeps that repository's `merged/<repo>`
  marker and `delivery_repo_landed` row, and its pass answers without touching the second.
- The three purity gates fail on a planted action importing the store or an HTTP client, a `Visit` field that is a list,
  and an action constructed with a collaborator; the layer walker fails a planted `hub/domain/actions/` import of
  `execution`.
- The kill-9 sweep reaches every point in [Crash points](./actions.md#crash-points), and the two-repository `land`
  killed between its repositories lands each exactly once on restart.
- End-to-end and crash tiers drive hub nodes through `test.answer` and `test.sequence`, which a hub without both fences
  refuses at mint as unknown actions.
- No hub process starts a child process for a graph: the hub's tests assert `subprocess` is never imported under
  `hub/delivery/`.
- `blizzard-mock`'s seeded graph routes every answer of `land`.
