# The flow record

## The tables

Every table here is tenant-owned (`tenant_id` foreign key, outside `GLOBAL_TABLES`) in `hub/store/schema.py`, read and
written through `ScopedConnections`. None is a configured record: a flow is made by the work that reaches it, never by
an operator, so it has no revision, no lifecycle, and no change log.

```text
flows
  flow_id        String       NOT NULL   flw_<ulid>, minted on the first submission to its resource
  tenant_id      String       NOT NULL
  resource       String       NOT NULL   repo:<repository_id>/<branch>
  repository_id  String       NOT NULL   the repository record's id
  branch         String       NOT NULL   the branch the flow lands on
  created_at     UtcDateTime  NOT NULL
  PRIMARY KEY (flow_id)
  UNIQUE (tenant_id, resource)

flow_submissions                          one per hub-node visit that reaches the hub
  submission_id  String       NOT NULL   fsub_<ulid>
  tenant_id      String       NOT NULL
  chunk_id       String       NOT NULL   FK chunks
  node_id        String       NOT NULL
  epoch          Integer      NOT NULL   the visit's epoch
  arrival        Integer      NOT NULL   the tenant's arrival sequence, assigned under the tenant row lock
  submitted_at   UtcDateTime  NOT NULL
  PRIMARY KEY (submission_id)
  UNIQUE (tenant_id, chunk_id, node_id, epoch)
  UNIQUE (tenant_id, arrival)

flow_submission_flows                     the flows a submission holds; none for a step that holds nothing
  submission_id, flow_id, tenant_id
  PRIMARY KEY (submission_id, flow_id)
  INDEX (tenant_id, flow_id)

flow_submission_closures                  append-only; at most one per submission
  submission_id  String       NOT NULL   FK flow_submissions
  tenant_id      String       NOT NULL
  outcome        String       NOT NULL   the verdict's choice, or stopped | restarted | completed | migrated | superseded
  closed_at      UtcDateTime  NOT NULL
  PRIMARY KEY (submission_id)

hub_turns                                 replaces hub_exec_slot
  turn_id        String       NOT NULL   htn_<ulid>
  tenant_id      String       NOT NULL
  submission_id  String       NOT NULL   FK flow_submissions
  started_at     UtcDateTime  NOT NULL
  ended_at       UtcDateTime  NULL       null while the turn runs
  outcome        String       NULL       the answered choice, pending, escalated, or interrupted
  PRIMARY KEY (turn_id)
  INDEX (tenant_id, submission_id)
  INDEX (tenant_id, ended_at)

flow_facts                                append-only; the flow's record
  id             Integer      NOT NULL
  tenant_id      String       NOT NULL
  flow_id        String       NOT NULL   FK flows
  submission_id  String       NOT NULL   FK flow_submissions
  chunk_id       String       NOT NULL
  kind           String       NOT NULL   submitted | turn-started | turn-ended | landed | withdrawn
  outcome        String       NULL       turn-ended: the turn's outcome; withdrawn: stopped | restarted | completed | migrated | superseded
  revision       String       NULL       landed: the revision the branch now carries
  at             UtcDateTime  NOT NULL
  INDEX (tenant_id, flow_id, id)
  INDEX (tenant_id, submission_id)
```

`IdPrefix` (`foundation/ids.py`) gains `FLOW = "flw"`, `FLOW_SUBMISSION = "fsub"`, and `HUB_TURN = "htn"`, and loses
`HUB_EXEC_SLOT` with the table it named. The arrival sequence is the newest `arrival` in the tenant plus one, read and
written under the tenant row lock — a tenant-wide read-then-insert, the shape `next_route_seq`
(`hub/store/internal/chunk_rows.py`) gives `route_created`'s per-chunk `seq` under `lock_chunk_row`. No sequence spans a
tenant today; `work_item_sequence` counts per work source. Domain types live in `hub/domain/delivery/flows.py` — `Flow`,
`Submission`, `Turn`, `FlowFact`, and the `FlowLine` that answers who is next ([turns.md](./turns.md#who-is-next)) from
a flow's open submissions — and the store in `hub/store/internal/flow_store.py`, behind `IReadFlows` and `IWriteFlows`
ports in `hub/domain/delivery/ports/`.

Whether a flow is busy, who is in its line, and who is landing are all derived from these rows (`bzh:facts-not-status`),
never stored: a flow is busy while a turn of a submission it holds has no `ended_at`, and a submission is open while it
has no closure. A closure is written once, in the transaction that ends the submission: a verdict
([verdict.md](./verdict.md#the-verdict)), a lever's withdrawal ([levers.md](./levers.md)), or the reconcile's
supersession ([verdict.md](./verdict.md#the-reconcile)). Every read the driver makes each tick — open submissions per
tenant, the flows each holds, open turns, the newest poll — is served by an index above or an existing one, so
`IReadFlows` needs no entry in the store-read-index gate's allow-list (`tests/store_scan_allowlist.py`).

`flow_facts` and the closures are kept as long as chunk facts are; bounding how long the hub keeps either is
`epic:retention`'s.

## What the record says

A flow's facts answer "what happened on `master` last night" in one place: who submitted, in what order, each turn taken
and how it ended, each landing and the revision it left, and each withdrawal and why. A submission's facts are written
to every flow it holds, so each flow's record stands alone.

A `landed` fact is written when a turn persists a `merged/<repo>` marker for the flow's repository. Under the actions
epic an action records that marker through its `IEffectSink` and the driver writes it through
`HubNodeExecutor.record_marker`, which today writes the marker (`record_hub_artifact`) and then, in a second
transaction, its `delivery_repo_landed` row. This epic makes the four writes one transaction: the marker, the
`delivery_repo_landed` row, the `landed` fact on that repository's flow with the marker's revision, and the
`flow-landed` event row ([below](#the-landed-event)). A crash can therefore never leave a landing marked but
unannounced, and a replay that finds the marker written finds the rest written with it.

A marker is keyed by `owner/repo`, which names no record; it reaches its flow through the repository id the visit
resolved for it, as `delivery_repo_landed.repository_id` is stamped today. Since each commit pointer resolves to its own
record, one `owner/repo` can stand for two records on two forges; a visit whose resolved repositories share an
`owner/repo` is refused at submit, routing `failure` as `_refuse` does — today the land script answers `failure` for it
(`land_common.py`) and the marker route raises `AmbiguousMarkerRepository` (`hub/delivery/marker_auth.py`) — so every
marker a submitted visit writes names one flow.

## The landed event

A `landed` fact is the event the initiative joins delivery to whatever follows it by. Its write records an `event_log`
row of the new kind `flow-landed`, severity info, with `detail`
`{flow_id, resource, repository_id, branch, revision, chunk_id, project_id}`, in the landing's own transaction, and
publishes it live through the broker once that transaction commits. `project_id` is the chunk's project, so a listener
knows whose work landed without reading the chunk. Nothing in this epic reacts to the event beyond the board.

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `flow-landed` (info) and `turn-interrupted`
(warning, [turns.md](./turns.md#a-turn-that-cannot-end)); `tests/event_log_kind_census.py` gains both, and
`blizzard-context:/domain/operations.md`'s table gains a row for each.

## Invariants

`tools/invariants.py`'s `OneLiveExecSlot` is replaced by three checks:

- `hub:one-turn-per-flow` — no flow has two open turns.
- `hub:one-turn-per-visit` — no submission has two open turns.
- `hub:flow-line-standing` — every open submission's chunk stands at its node, at its epoch, or at a later one while the
  chunk is `needs_human`, so no visit that has left a node still holds a place.

The trace corpus follows the table: `TracedHubExecSlot` (`hub/domain/observability/tracing/facts.py`, loaded by
`hub/store/internal/corpus_rows.py`'s `_slots`) becomes `TracedHubTurn`, `tracing/summary.py` derives its `HUB_EXEC`
intervals from turns, and the `HUB_EXEC` span is keyed by the turn id where it was keyed by the slot's. The slot's rows
are dropped with its table, so a chunk traced before the upgrade loses its `HUB_EXEC` intervals; the rest of its trace
stands.
