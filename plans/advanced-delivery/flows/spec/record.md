# The flow record

## The tables

Every table here is tenant-owned (`tenant_id` foreign key, outside `GLOBAL_TABLES`) in `hub/store/schema.py`, read and
written through `ScopedConnections`. None is a configured record: a flow is made by the work that reaches it, never by
an operator, so it has no revision, no lifecycle, and no change log.

```text
flows
  flow_id        String       NOT NULL   flw_<ulid>, minted on the first submission to its resource; hub_unique
  tenant_id      String       NOT NULL
  resource       String       NOT NULL   repo:<repository_id>/<branch>
  repository_id  String       NOT NULL   the repository record's id
  branch         String       NOT NULL   the branch the flow lands on
  created_at     UtcDateTime  NOT NULL
  PRIMARY KEY (flow_id)
  UNIQUE (tenant_id, resource)

flow_submissions                          one per hub-node visit that reaches the hub
  submission_id  String       NOT NULL   fsub_<ulid>; hub_unique
  tenant_id      String       NOT NULL
  chunk_id       String       NOT NULL   FK chunks
  node_id        String       NOT NULL
  epoch          Integer      NOT NULL   the visit's epoch
  generation     Integer      NOT NULL   0; one more for each resubmission of the same visit
  arrival        Integer      NOT NULL   the tenant's arrival sequence, assigned under the tenant row lock
  submitted_at   UtcDateTime  NOT NULL
  PRIMARY KEY (submission_id)
  UNIQUE (tenant_id, chunk_id, node_id, epoch, generation)
  UNIQUE (tenant_id, arrival)

flow_submission_flows                     the flows a submission holds; none for a step that holds nothing
  submission_id, flow_id, tenant_id
  PRIMARY KEY (submission_id, flow_id)
  INDEX (tenant_id, flow_id)

flow_submission_closures                  append-only; at most one per submission
  submission_id  String       NOT NULL   FK flow_submissions
  tenant_id      String       NOT NULL
  outcome        String       NOT NULL   the verdict's choice, or stopped | restarted | completed | superseded
  closed_at      UtcDateTime  NOT NULL
  PRIMARY KEY (submission_id)
  INDEX (tenant_id, submission_id)

hub_turns                                 replaces hub_exec_slot
  turn_id        String       NOT NULL   htn_<ulid>; hub_unique
  tenant_id      String       NOT NULL
  submission_id  String       NOT NULL   FK flow_submissions
  started_at     UtcDateTime  NOT NULL
  ended_at       UtcDateTime  NULL       null while the turn runs
  outcome        String       NULL       the answered choice, a reserved wait, escalated, superseded, cut-off, or interrupted
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
  outcome        String       NULL       turn-ended: the turn's outcome; withdrawn: stopped | restarted | completed | superseded
  revision       String       NULL       landed: the revision the branch now carries
  at             UtcDateTime  NOT NULL
  INDEX (tenant_id, flow_id, id)
  INDEX (tenant_id, submission_id)
```

Each minted string key — `flow_id`, `submission_id`, `turn_id` — is declared `info={"hub_unique": True}` in
`hub/store/schema.py`, and every other key is either one of those, a foreign key onto one, or led by `tenant_id`, so the
tenant schema gate (`tests/test_tenant_schema_gate.py`) finds no natural key two tenants could both claim and no table
without a tenant-led index. `flow_facts.id` is the table's autoincrement surrogate, which the gate already admits.

A visit almost always has one submission. It has a second only when a turn found its resources changed under it — an
operator edited a repository's base branch or a project's links while it waited — and closed the first `superseded`
([turns.md](./turns.md#what-a-step-holds)); the reconcile then submits it again at the next `generation`. The reconcile
submits a visit only when it has no open submission, under the tenant row lock, and the unique key refuses a second
insert of the same generation, so a replayed completion, a repeated hub-advance, or two ticks seeing the same chunk
still never queue it twice.

A turn belongs to exactly one submission in this epic. The `trains` epic widens it to a batch — `submission_id` becomes
nullable beside a `batch_id`, with a check that exactly one is set — and nothing here assumes the column can never be
null beyond this epic's own reads.

`IdPrefix` (`foundation/ids.py`) gains `FLOW = "flw"`, `FLOW_SUBMISSION = "fsub"`, and `HUB_TURN = "htn"`, and loses
`HUB_EXEC_SLOT` with the table it named. The arrival sequence is the newest `arrival` in the tenant plus one, read and
written under the tenant row lock — a tenant-wide read-then-insert, the shape `next_route_seq`
(`hub/store/internal/chunk_rows.py`) gives `route_created`'s per-chunk `seq` under `lock_chunk_row`. No sequence spans a
tenant today; `work_item_sequence` counts per work source. Domain types live in `hub/domain/delivery/flows.py` — `Flow`,
`Submission`, `Turn`, `FlowFact`, and the `FlowLine` that answers who is next ([turns.md](./turns.md#who-is-next)) from
a flow's open submissions — and the store in `hub/store/internal/flow_store.py`, behind `IReadFlows` and `IWriteFlows`
ports in `hub/domain/delivery/ports/`.

Whether a flow is busy, who is in its line, and who is landing are all derived from these rows (`bzh:facts-not-status`),
never stored. A flow is busy while any turn of any submission it holds has no `ended_at`, read from `hub_turns` joined
to `flow_submission_flows` whether or not the submission is still open; it is never derived by way of open submissions,
since a submission must not be able to free its branch by closing while its pass still runs. A submission is open while
it has no closure.

A closure is written once, in the transaction that ends the submission: a verdict
([verdict.md](./verdict.md#the-verdict)), a lever's withdrawal ([levers.md](./levers.md)), a turn that found its
resources changed ([turns.md](./turns.md#what-a-step-holds)), or the reconcile's supersession
([verdict.md](./verdict.md#the-reconcile)). No writer closes a submission while it has an open turn, save the
transaction that ends that turn: a lever or the reconcile that finds a turn open leaves the closure to the turn's end,
which writes it beside `turn-ended`. Which outcome a withdrawal records is decided in one place,
`withdrawal_for(chunk, submission)` (`hub/domain/delivery/flows.py`): `stopped` or `completed` for a chunk that ended by
that lever, `restarted` for one restarted past the visit, `superseded` for any other move off the visit, and nothing
while the visit is current or a person holds the chunk at a later epoch. Every writer that withdraws asks it — a lever,
a turn's end, the in-tick sweep, boot recovery, and the reconcile — so a chunk stopped mid-turn is recorded `stopped`
whichever of them reaches it. Several writers can still race to close a submission with no open turn, so every closure
is an insert made only when no closure exists, checked inside the writer's transaction under the tenant row lock: the
first to commit wins, its outcome is the one recorded, and a later writer finds the closure and writes neither a second
closure nor a second `withdrawn` fact.

Every read the driver makes each tick — the open submissions from the tenant's arrival floor
([turns.md](./turns.md#the-flow-driver)), the flows each holds, open turns, the newest poll — is served by an index
above or an existing one, so `IReadFlows` needs no entry in the store-read-index gate's allow-list
(`tests/store_scan_allowlist.py`), whose entries for the slot's scans are removed with the slot. Each `IReadFlows`
method gains its recipe in `tests/store_read_census.py`, which `test_hub_census_is_exhaustive` requires of every
`IRead*` method.

`flow_facts` and the closures are kept as long as chunk facts are; bounding how long the hub keeps either is
`epic:retention`'s.

## What the record says

A flow's facts answer "what happened on `master` last night" in one place: who submitted, in what order, each turn taken
and how it ended, each landing and the revision it left, and each withdrawal and why. A submission's facts are written
to every flow it holds, so each flow's record stands alone.

A `landed` fact is written when a turn persists a `merged/<repo>` marker for the flow's repository. Under the actions
epic an action hands that marker over as a `LandingRecord` effect, which the writer admits past the chunk's fence — a
stopped or completed chunk, or a stale epoch — because it records a merge the forge has already made, and writes with
its `delivery_repo_landed` row and its close intents. This epic widens that admitted transaction to five writes: the
marker, the `delivery_repo_landed` row, the close intents, the `landed` fact on that repository's flow with the marker's
revision, and the `flow-landed` event row ([below](#the-landed-event)). A crash can therefore never leave a landing
marked but unannounced, a replay that finds the marker written finds the rest written with it, and a merge made by a
chunk stopped mid-pass is on its flow's record like any other ([levers.md](./levers.md)).

A marker is keyed by `owner/repo`, which names no record; it reaches its flow through the repository id the visit
resolved for it, as `delivery_repo_landed.repository_id` is stamped today. Since each commit pointer resolves to its own
record, one `owner/repo` can stand for two records on two forges; a visit whose resolved repositories share an
`owner/repo` is refused at submit, routing `failure` as `_refuse` does — today the land script answers `failure` for it
(`land_common.py`) and the marker route raises `AmbiguousMarkerRepository` (`hub/delivery/marker_auth.py`) — so every
marker a submitted visit writes names one flow.

## The landed event

A `landed` fact is the event the initiative joins delivery to whatever follows it by. Its write records an `event_log`
row of the new kind `flow-landed`, severity info, with `detail`
`{flow_id, resource, repository_id, branch, revision, chunk_id, project_id, members}`, in the landing's own transaction,
and publishes it live through the broker once that transaction commits. `project_id` is the chunk's project, so a
listener knows whose work landed without reading the chunk. `members` lists every chunk whose work the landing carried,
each as `{chunk_id, project_id}`; in this epic a landing carries one chunk, so `members` holds the one `chunk_id` and
`project_id` the detail already names. It is there from the first event so that a landing carrying several chunks, which
the `trains` epic brings, lengthens the list rather than changing the event's shape. Nothing in this epic reacts to the
event beyond the board.

The record starts at the upgrade. A repository a chunk landed before it carries its marker and its
`delivery_repo_landed` row, which the board and the chunk's delivery rows keep reading as they do today, but no `landed`
fact and no `flow-landed` event: the migration cannot tell which branch a past landing went to, since the repository's
`base_branch` may have changed since, and a flow's record does not claim what it did not see.

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `flow-landed` (info) and `turn-interrupted`
(warning, [turns.md](./turns.md#a-turn-that-cannot-end)); `tests/event_log_kind_census.py` gains both, and
`blizzard-context:/domain/operations.md`'s table gains a row for each.

## Invariants

`tools/invariants.py`'s `OneLiveExecSlot` is replaced by three checks:

- `hub:one-turn-per-flow` — no flow has two open turns.
- `hub:one-turn-per-visit` — no submission has two open turns.
- `hub:flow-line-standing` — every open submission's chunk stands at its node, at its epoch, or at a later one while the
  chunk is `needs_human` or `paused`, so no visit that has left a node still holds a place. Two kinds of submission are
  exempt, because the line is converging on them by design: one with an open turn, whose closure waits for the turn's
  end, and one whose chunk moved less than a minute ago, the longest a full reconcile is ever skipped. With those exempt
  the check holds at any moment, so `blizzard dev` may run it against a live store as the crash sweep and the scenarios
  do once the hub is quiescent.

The trace corpus follows the table: `TracedHubExecSlot` (`hub/domain/observability/tracing/facts.py`, loaded by
`hub/store/internal/corpus_rows.py`'s `_slots`) becomes `TracedHubTurn`, `tracing/summary.py` derives its `HUB_EXEC`
intervals from turns, and the `HUB_EXEC` span is keyed by the turn id where it was keyed by the slot's. The slot's rows
are dropped with its table, so a chunk traced before the upgrade loses its `HUB_EXEC` intervals; the rest of its trace
stands.
