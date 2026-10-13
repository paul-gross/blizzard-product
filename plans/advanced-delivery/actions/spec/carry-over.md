# Carry-over

An installation keeps landing exactly as it did, with every stored graph, every pinned chunk, and every chunk
mid-delivery carried onto actions in one release, and nothing re-pinned or re-minted by hand.

## Translation

Every `run:` blizzard has shipped is one step running one packaged script, so each maps to one action.
`LEGACY_RUN_COMMANDS` (`hub/domain/graph/legacy_run.py`) holds the map:

| A `run:` whose one step's command is                                                         | Becomes                                              |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `python3 -m blizzard.hub.graphs.scripts.land_pr_ci`                                          | `action: land`                                       |
| `python3 -m blizzard.hub.graphs.scripts.review_deliver`                                      | `action: record-findings`                            |
| `python3 -m blizzard.hub.graphs.scripts.garden_deliver --delta <delta> --proposals <docket>` | `action: garden-deliver`, `with: {delta, proposals}` |
| `python3 -m blizzard.hub.graphs.scripts.land_default`                                        | `action: land`                                       |
| `python3 -m blizzard.hub.graphs.scripts.land_ff`                                             | `action: land`                                       |

The last two rows are scripts packaged graphs once named and no longer do: `land_default` was the default graph's
delivery until it became a triage router, and `land_ff` fast-forwarded the basic lanes until they moved to `land_pr_ci`.
Stored versions minted from them remain in every tenant that ran those graphs. Both carry onto `land`, the one landing
the hub keeps: `land_default` opened and merged a pull request as `land` does, only without waiting on checks, and
`land_ff`'s module is already gone, so a chunk pinned to a version naming it cannot land today and lands through a pull
request once carried over.

The map is used in two places. `GraphDoc.of` translates a `run:` it matches into its action, so a stored
`definition_yaml` written before the slice still parses — `graph_sync` compares against it, and re-exporting an old
version yields a document that mints. The migration translates the stored nodes.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, frozen once released
(`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`). For every `graph_nodes` row in every
tenant it sets `action` and `action_args` from `run` through the map, then drops `run`. A row whose `run` the map does
not match keeps no action and is reported by name, graph, version, and tenant in the migration's log; `definition_yaml`
keeps the command for the record. Because `GraphDoc.of` translates a stored `definition_yaml` the same way and
`GraphDoc.differs_from` compares parsed documents, `graph_sync` finds each packaged graph equal to its newest stored
version and mints nothing, so nothing is re-pinned.

## What does not translate

A node left with no action never runs. A chunk that reaches it, or sits at it when the hub upgrades, takes no turn: the
executor records an escalation (`EscalationCause.HUB_ACTION_MISSING`) naming the node and the command its graph's
`definition_yaml` carried, and a `hub-action-missing` event (critical), so the chunk is `needs_human` and its operator
restarts or migrates it onto a graph that names an action. `EventLogKind`, `EVENT_LOG_SEVERITY`,
`tests/event_log_kind_census.py` at the repository root, and `blizzard-context:/domain/operations.md` gain the kind.
`tools/invariants.py` gains `hub:hub-node-has-action`: every hub node at a version a live chunk is pinned to names an
action.

## What is removed

- `IHubCommandRunner` and `CommandResult` (`hub/delivery/command_runner.py`), `SubprocessHubCommandRunner`
  (`hub/delivery/internal/hub_command_runner.py`), and their wiring in `hub/composition.py`.
- The per-chunk working directory a step ran in: `IHubWorkdir` (`hub/delivery/workdir.py`), `FilesystemHubWorkdir`
  (`hub/delivery/internal/hub_workdir.py`), and their wiring.
- `HubEnv`, every `ENV_*` constant, `PrintedChoice`, `_run_step`, `_NoopStep`, `FailedStep`, and the three `hubnode.*`
  crash points around a step (`hub/delivery/hub_node.py`). The `bzh:hub-node-env-contract` variables are no longer set
  anywhere. `BounceEnvelope.step`, which carried the `FailedStep`, becomes `action`, the name of the action whose pass
  the bounce came from.
- The step's trace span, `RUN_STEP_SPAN` and its `RUN_STEP_*` attributes
  (`hub/domain/observability/tracing/platform.py`), and their description in `docs/deployment/tracing.md`; a pass is
  traced under the visit's `HUB_EXEC` span.
- The marker token: `MarkerAuthority`'s minting and verification (`hub/delivery/marker_auth.py`),
  `require_marker_authority` (`hub/api/marker_auth.py`), and the three routes it guarded —
  `POST /api/chunks/{chunk_id}/hub-markers`, `/garden-delivery`, and `/review-findings-delivery` (`hub/api/chunks.py`) —
  with their wire models, and `blizzard hub record-marker` (`hub/cli/marker.py`, its entry in `hub/cli/__init__.py`, and
  its place in `cli/operator_trace.py`'s `_NEVER_TRACED`), which existed for a script to call the first. The
  `garden-delivery-failure` and `review-findings-failure` markers the scripts wrote are written by their actions under
  the same names, so the prompts that read them are unchanged. The visit's resolved repositories, which
  `hold_repositories` pinned for the marker route, reach the action on the `Visit`, and
  `delivery_repo_landed.repository_id` is stamped from them.
- `hub/graphs/scripts/` and its tests, whose cases move onto the actions.

The `BZ_FORGE_*` keys in `hub/domain/config/legacy_keys.py` stay: they are the legacy configuration the hub refuses to
start with and imports through `blizzard hub config import-legacy`, not variables handed to a step.

No runner changes: nothing here crosses the runner's wire.

## Proof

- Every shipped graph's hub node, minted from its packaged file and from each stored version's `definition_yaml`, names
  the action its script was.
- A fixture store captured before the migration, with chunks pinned to old graph versions and one mid-`pending` on
  `land`, carries over and lands every chunk against the mock forge with the outcomes and markers the scripts produced.
- `record-findings` and `garden-deliver` record what their scripts recorded, and answer `invalid` where they did.
- A graph naming `run:`, an unknown action, or the wrong `with:` keys is refused at mint.
- A node left without an action escalates its chunk with the command named, and `hub:hub-node-has-action` reports it.
- No hub process starts a child process for a graph: the hub's tests assert `subprocess` is never imported under
  `hub/delivery/`.
