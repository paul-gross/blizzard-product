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

hub_turns                                 replaces hub_exec_slot
  turn_id        String       NOT NULL   htn_<ulid>
  tenant_id      String       NOT NULL
  submission_id  String       NOT NULL   FK flow_submissions
  started_at     UtcDateTime  NOT NULL
  ended_at       UtcDateTime  NULL       null while the turn runs
  outcome        String       NULL       the answered choice, pending, refused, escalated, or interrupted
  PRIMARY KEY (turn_id)

flow_facts                                append-only; the flow's record
  id             Integer      NOT NULL
  tenant_id      String       NOT NULL
  flow_id        String       NOT NULL   FK flows
  submission_id  String       NOT NULL   FK flow_submissions
  chunk_id       String       NOT NULL
  kind           String       NOT NULL   submitted | turn-started | turn-ended | landed | withdrawn
  outcome        String       NULL       turn-ended: the turn's outcome; withdrawn: stopped | restarted | completed | migrated
  revision       String       NULL       landed: the revision the branch now carries
  at             UtcDateTime  NOT NULL
  INDEX (tenant_id, flow_id, id)
```

`IdPrefix` (`foundation/ids.py`) gains `FLOW = "flw"`, `FLOW_SUBMISSION = "fsub"`, and `HUB_TURN = "htn"`. The arrival
sequence is the newest `arrival` in the tenant plus one, read and written under the tenant row lock — a tenant-wide
read-then-insert, the shape `next_route_seq` (`hub/store/internal/chunk_rows.py`) gives `route_created`'s per-chunk
`seq` under `lock_chunk_row`. Nothing keeps a tenant-wide sequence today. Domain types live in
`hub/domain/delivery/flows.py` — `Flow`, `Submission`, `Turn`, `FlowFact`, and the `FlowLine` that answers who is next
([turns.md](./turns.md#who-is-next)) from a flow's open submissions — and the store in
`hub/store/internal/flow_store.py`, behind `IReadFlows` and `IWriteFlows` ports in `hub/domain/delivery/ports/`.

Whether a flow is busy, who is in its line, and who is landing are all derived from these rows (`bzh:facts-not-status`),
never stored: a flow is busy while a turn of a submission it holds has no `ended_at`, and a submission is open until a
`withdrawn` fact or its visit's turn ends with a verdict ([verdict.md](./verdict.md)).

## What the record says

A flow's facts answer "what happened on `master` last night" in one place: who submitted, in what order, each turn taken
and how it ended, each landing and the revision it left, and each withdrawal and why. A submission's facts are written
to every flow it holds, so each flow's record stands alone. A `landed` fact is written when a turn records a
`merged/<repo>` marker for the flow's repository: `HubNodeExecutor.record_marker` already appends `delivery_repo_landed`
there, and it now appends the fact to that repository's flow in the same transaction, with the marker's revision. A
marker is keyed by `owner/repo`, which names no record; it reaches its flow through the repository id the visit resolved
for it, as `delivery_repo_landed.repository_id` is stamped today. Since each commit pointer resolves to its own record,
one `owner/repo` can stand for two records on two forges; a visit whose resolved repositories share an `owner/repo` is
refused at submit, routing `failure` as `_refuse` does — today the land script answers `failure` for it
(`land_common.py`) and the marker route raises `AmbiguousMarkerRepository` (`hub/delivery/marker_auth.py`) — so every
marker a submitted visit writes names one flow.

## The landed event

A `landed` fact is the event the epic joins delivery to what follows it by. Its write records an `event_log` row of the
new kind `flow-landed`, severity info, with `detail` `{flow_id, resource, repository_id, branch, revision, chunk_id}`,
through `EventLogService.record`, which also publishes it live. `epic:advanced-deployment` consumes the fact; nothing in
this slice reacts to it beyond the board.

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `flow-landed` (info) and `turn-interrupted`
(warning, [carry-over.md](./carry-over.md)); `tests/event_log_kind_census.py` gains both, and
`blizzard-context:/domain/operations.md`'s table gains a row for each.

## Invariants

`tools/invariants.py`'s `OneLiveExecSlot` is replaced by three checks:

- `hub:one-turn-per-flow` — no flow has two open turns.
- `hub:one-turn-per-visit` — no submission has two open turns.
- `hub:flow-line-standing` — every open submission's chunk is at its node and epoch, so no withdrawn visit still holds a
  place.

The trace corpus follows the table: `TracedHubExecSlot` (`hub/domain/observability/tracing/facts.py`, loaded by
`hub/store/internal/corpus_rows.py`'s `_slots`) becomes `TracedHubTurn`, `tracing/summary.py` derives its `HUB_EXEC`
intervals from turns, and the `HUB_EXEC` span is keyed by the turn id where it was keyed by the slot's.
