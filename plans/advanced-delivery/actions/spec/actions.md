# Actions

## The node

A hub node names an action in place of `run:`:

```yaml
deliver:
  executor: hub
  action: land
  poll_interval: 30
  poll_timeout: 1800
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
`NodeShape.is_hub_command_node` becomes `executor is HUB and action`. The graph wire model, `GraphNodeView`
(`wire/graph.py`), carries no command today; it gains `action` and `action_args`, the OpenAPI document with it, and the
web graph view shows a hub node's action.

`validation.py`'s `_check_run` becomes `_check_action`: `action` is legal only on `executor: hub` and required there; it
names a registered action; `with` carries exactly the arguments that action declares. `bounce_cap`, `poll_interval`,
`poll_timeout`, the authored `failure` choice, and every other rule a hub node is held to are unchanged. `run:` is
refused at mint, except as [carry-over.md](./carry-over.md#translation) translates it.

## The registry

`HUB_ACTIONS` (`hub/delivery/actions/__init__.py`) is the closed set, keyed by name. An action implements `IHubAction`
(`hub/delivery/actions/base.py`):

```python
class IHubAction(Protocol):
    name: str
    args: frozenset[str]          # the keys `with:` must carry
    lands: bool                   # whether a visit holds the branches its chunk lands on (flows slice)
    async def take(self, visit: Visit) -> Pass: ...
```

`Visit` is everything the `run:` environment carried, as values rather than variables: the chunk, its graph and node,
the epoch, the node's `action_args`, the chunk's artifacts, its `GitCommits`, its `RepositoryResolution`
(`hub/domain/config/repositories.py`) — one `ResolvedRepository` per commit pointer, each with its own forge, owner,
base branch, and secret, since commit pointers resolve per repository — the work items, the feature title, the chunk's
public URL, whether the node expects git commits, the project's id and slug, and a `MarkerWriter` that records a marker
through `HubNodeExecutor.record_marker` directly. It holds no secret: an action that needs a repository's credential
asks for it through the `ISecretReader` it was built with, and holds it no longer than its pass.

`Pass(choice, log)` answers with an authored choice or `pending`, and the lines that today reach a step's
`hub-log.<step>` artifact. `HubNodeExecutor._run_locked` keeps everything before and after the step loop — the
poll-timeout check, resolution and its refusals, routing, bounces — and replaces the loop with one `take`, run on the
hub's event loop under `BZ_HUB_ACTION_PASS_SECONDS`, 60 by default. Every caller of the executor is synchronous today —
`HubNodeExecutor.run`, `IHubNodeExecutor.run` (`hub/domain/execution/apply.py`), and the `hub-advance` route, a plain
`def` FastAPI runs on a worker thread — so in this slice the executor reaches the event loop through
`anyio.from_thread.run` and awaits the pass there; the `flows` slice's driver awaits `take` directly and retires the
bridge. A pass that overruns is cancelled and answers `pending`, so the poll timeout, not the pass bound, decides when
waiting has gone on too long. An exception escaping a pass answers `failure` with its traceback in the
`hub-log.<action>` artifact, as a crashed script does today; the artifact a step wrote is named for the step
(`hub-log.land-every-repo`), so the `resolve` prompt of `advanced-development-workflow` that reads it reads
`hub-log.land` instead. Every forge call an action makes carries a timeout of its own.

Each action keeps its own idempotency, as the scripts did: a pass re-reads the markers its visit already wrote and skips
what they say is done. The step-level `produces:` skip goes with the step.

## The three actions

| Action            | Arguments            | Answers                                                         | Lands | From                                                    |
| ----------------- | -------------------- | --------------------------------------------------------------- | ----- | ------------------------------------------------------- |
| `land`            | —                    | `landed`, `conflict`, `failure`, `inherited-failure`, `pending` | yes   | `land_pr_ci.py`, `land_common.py`, `base_merge.py`      |
| `record-findings` | —                    | `recorded`, `invalid`                                           | no    | `review_deliver.py` → `services.review_findings.record` |
| `garden-deliver`  | `delta`, `proposals` | `recorded`, `invalid`                                           | no    | `garden_deliver.py` → `services.garden_delivery.record` |

**`land`** (`hub/delivery/actions/land.py`) is `land_pr_ci` moved into the hub with its stages, outcomes, and markers
unchanged — `merged/<repo>`, `delivery-pr/<repo>/<n>`, `delivery-findings`, `delivery-findings/foreign-head`,
`ci-rerun/<…>`. Its GitHub calls move with it into `hub/delivery/actions/internal/github_client.py`, an `httpx` client
built per pass for each resolved repository from that repository's own forge address and revealed credential, as
`LandRun.at` binds each commit to its own target today; the `forges` slice replaces that client with the landing seam.
`BZ_HUB_LAND_TEST_PAUSE_SECONDS` becomes an injected pause the tests set.

**`record-findings`** and **`garden-deliver`** call the services their scripts reached by HTTP, with the visit's chunk,
node, and epoch, and answer `invalid` where the route answered it. A fenced write answers `failure`, as the script's
non-zero exit did.

`land_default.py`, which no packaged graph names since the default graph became a triage router, is deleted with the
other scripts and `hub/graphs/scripts/`; stored versions that still name it are carried onto `land`
([carry-over.md](./carry-over.md#translation)).
