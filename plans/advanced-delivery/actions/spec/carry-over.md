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

`garden_deliver`'s arguments are repeatable flags, so its row is a parse, not a lookup: the command is split as a shell
would split it, each `--delta` and `--proposals` value is gathered into its argument's list, and a command carrying any
other flag does not translate.

The `land_default` and `land_ff` rows are scripts packaged graphs once named and no longer do: `land_default` was the
default graph's delivery until it became a triage router, and `land_ff` fast-forwarded the basic lanes until they moved
to `land_pr_ci`. Stored versions minted from them remain in every tenant that ran those graphs, and both carry onto
`land`, the one landing the hub keeps. Neither lands exactly as it did. `land_default` merged a pull request GitHub
called clean without reading its checks; under `land` the chunk waits on them, and a repository with no CI waits until
the poll timeout. `land_ff`'s module is already gone, so a chunk pinned to a version naming it cannot land today and
lands through a pull request once carried over. Both kinds of version author only `landed` and `conflict`, so when
`land` answers `failure` or `inherited-failure` for one, the chunk goes to a person
([Routing an answer](./actions.md#routing-an-answer)).

The map lives in `LEGACY_RUN_COMMANDS` (`hub/domain/graph/legacy_run.py`) and is used where a stored document is read:
`graph_sync`'s `_parsed` and a stored version's export translate a `run:` it matches into its action, so a
`definition_yaml` written before the slice still parses, `graph_sync` compares against it, and re-exporting an old
version yields a document that mints. A graph authored from here on is never translated; `run:` in it is refused. The
migration restates the map in its own frozen revision rather than importing `legacy_run.py`.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, frozen once released
(`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`). For every hub `graph_nodes` row in every
tenant it sets `action` and `action_args` from `run` through the map, then drops `run`. A row whose `run` the map does
not match keeps no action and is reported by name, graph, version, and tenant in the migration's log; `definition_yaml`
keeps the command for the record.

`graph_sync` then mints one new version of each packaged graph, since this slice changes their `poll_timeout` and the
`advanced-development-workflow` prompts that name the landing log. Chunks already pinned stay on the version they hold;
only chunks minted after the release take the new one. `blizzard-mock`'s seeded graph (`graph_seed.py`) writes its
`graph_nodes` rows and its `definition_yaml` directly; both gain `action: land` on its `deliver` node.

## What does not translate

A node left with no action never runs, and nothing stalls on it unseen. Because every hub node is driven
([The node](./actions.md#the-node)), a chunk that reaches one, or that `hub-advance` finds standing at one, reaches the
executor, which records an escalation (`EscalationCause.HUB_ACTION_MISSING`) naming the node and the command its graph's
`definition_yaml` carried, and a `hub-action-missing` event (critical). A chunk already standing at such a node when the
hub upgrades, with no runner left to call `hub-advance` — a detached chunk, say — is escalated the same way by a pass
the hub makes at boot, after the migration, over every open tenant's live chunks. Either way the chunk is `needs_human`,
and its operator restarts or migrates it onto a graph that names an action.

`EscalationCause` (`foundation/escalation_causes.py`) gains `HUB_ACTION_MISSING` and `UNROUTABLE_OUTCOME`, the OpenAPI
enum with them, and the board's `CAUSE_SENTENCES` (`web/projects/fleet/src/lib/chunk/chunk-detail/chunk-escalation.ts`)
a sentence for each. `EventLogKind`, `EVENT_LOG_SEVERITY`, `tests/event_log_kind_census.py` at the repository root, and
`blizzard-context:/domain/operations.md` gain `hub-action-missing`, `unroutable-outcome`, and `hub-action-overran`.
`tools/invariants.py` gains `hub:hub-node-has-action`: every hub node at a version a live chunk is pinned to names an
action.

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
  ``"`{action}` answered {choice}"`` where it read the step's exit code. The crash points around a step give way to the
  effect writer's ([Crash points](./actions.md#crash-points)).
- The step's trace span, `RUN_STEP_SPAN` and its `RUN_STEP_*` attributes
  (`hub/domain/observability/tracing/platform.py`), and their description in `docs/deployment/tracing.md`; a pass is
  traced under the visit's `HUB_EXEC` span, which gains `blizzard.hub.action.name` and `blizzard.hub.action.choice`.
- The marker token: `MarkerAuthority`'s minting and verification (`hub/delivery/marker_auth.py`), the process-level
  `marker_authority` and `hub_marker_callback_base_url` composition builds it from (`hub/composition.py`, `hub/app.py`),
  `MarkerPrincipal`, `MARKER_TOKEN_HEADER`, and the marker branch of `_machine_credential` (`hub/api/auth_session.py`),
  `require_marker_authority` (`hub/api/marker_auth.py`), and the three routes it guarded —
  `POST /api/chunks/{chunk_id}/hub-markers`, `/garden-delivery`, and `/review-findings-delivery` (`hub/api/chunks.py`) —
  with their wire models, and `blizzard hub record-marker` (`hub/cli/marker.py`, its entry in `hub/cli/__init__.py`, and
  its place in `cli/operator_trace.py`'s `_NEVER_TRACED`), which existed for a script to call the first. Those routes
  also admitted a person holding `CHUNK_CONTROL`, and that operator path goes with them. The `garden-delivery-failure`
  and `review-findings-failure` markers the scripts wrote are written by their actions under the same names, so the
  prompts that read them are unchanged. The visit's resolved repositories, which `hold_repositories` pinned for the
  marker route, reach the action on the `Visit`, and `delivery_repo_landed.repository_id` is stamped from them.
- `BZ_HUB_LAND_TEST_PAUSE_SECONDS`: the kill-9 sweep reaches the windows it held open through armed crash points
  instead.
- `hub/graphs/scripts/` and its tests, whose cases move onto the actions.

The `BZ_FORGE_*` keys in `hub/domain/config/legacy_keys.py` stay: they are the legacy configuration the hub refuses to
start with and imports through `blizzard hub config import-legacy`, not variables handed to a step.

No runner changes: nothing here crosses the runner's wire.

## Documentation

`docs/` drops every mention of `run:`, and of what a step was handed even where `run:` is not named:
`docs/deployment/repositories.md` stops describing the `BZ_FORGE_*` variables as the land script's, `docs/backup.md`
stops listing hub working directories, and `docs/deployment/recovery.md` names the effect writer's crash points in place
of the `hubnode.*` ones. In `blizzard-context`, `standards/worker-nodes/declarations.md` stops citing `run-steps.md` and
`bzh:hub-node-run-shape`.

## Proof

- Every shipped graph's hub node, minted from its packaged file and from each stored version's `definition_yaml`, names
  the action its script was, and a node with an empty `run:` names `pass`.
- A fixture store captured before the migration, with chunks pinned to old graph versions and one mid-`pending` on
  `land`, carries over and lands every chunk against the mock forge with the outcomes and markers the scripts produced.
- `record-findings` and `garden-deliver` record what their scripts recorded, and answer `invalid` where they did.
- A graph naming `run:`, an unknown action, the wrong `with:` keys, or an action whose answer it leaves unrouted is
  refused at mint.
- A node left without an action escalates its chunk with the command named, whether the chunk reaches it, is driven at
  it by `hub-advance`, or stood at it, detached, when the hub upgraded; `hub:hub-node-has-action` reports it.
- A carried-over version that does not route the choice its pass answered escalates its chunk at once, naming the
  choice.
- A pass that overruns its bound answers `pending` and records `hub-action-overran`.
- The three purity gates fail on a planted action importing the store, a `Visit` field that is a list, and an action
  constructed with a collaborator.
- The kill-9 sweep reaches every point in [Crash points](./actions.md#crash-points), and the two-repository `land`
  killed between its repositories lands each exactly once on restart.
- End-to-end and crash tiers drive hub nodes through `test.answer` and `test.sequence`, which a hub without both fences
  refuses at mint as unknown actions.
- No hub process starts a child process for a graph: the hub's tests assert `subprocess` is never imported under
  `hub/delivery/`.
