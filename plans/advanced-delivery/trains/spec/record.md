# The train's record

## The tables

Every table here is tenant-owned and written through `ScopedConnections`, beside the flows slice's tables in
`hub/store/schema.py`. A batch is made by the work that reaches it, so, like a flow, it has no revision and no change
log.

```text
flow_batches
  batch_id          String       NOT NULL   fbt_<ulid>
  tenant_id         String       NOT NULL
  flow_id           String       NOT NULL   FK flows
  split_from        String       NULL       the batch this one was split from
  base_revision     String       NOT NULL   the base's tip the train was built on; changes on a rebuild
  train_ref         String       NOT NULL   <train_prefix><flow_id>/<batch_id>
  revision          String       NULL       the train's tip once every member is merged
  formed_at         UtcDateTime  NOT NULL
  PRIMARY KEY (batch_id)

flow_batch_members
  batch_id          String       NOT NULL   FK flow_batches
  submission_id     String       NOT NULL   FK flow_submissions
  tenant_id         String       NOT NULL
  position          Integer      NOT NULL   arrival order within the batch
  merged_revision   String       NULL       the train's tip once this member was merged in
  PRIMARY KEY (batch_id, submission_id)
```

`IdPrefix` (`foundation/ids.py`) gains `FLOW_BATCH = "fbt"`. Domain types join `hub/domain/delivery/flows.py` — `Batch`,
`BatchMember`, and `BatchState`, derived — and the store `flow_store.py`, behind the same ports.

`flow_facts` gains a nullable `batch_id`, and its `kind` gains `batch-formed`, `batch-built`, `batch-landed`,
`batch-split`, `batch-abandoned`, `left-batch`, and `ejected`; `outcome` carries the ejection's verdict or the reason a
batch was abandoned — `unproven`, `outpaced`, or `advance-refused`. A batch's state — building, proving, landed, split,
abandoned — is derived from its facts and its member rows (`bzh:facts-not-status`), never stored. A member's verdict is
the newest `landed` or `ejected` fact naming its submission.

## The train turn

A train's passes are turns on its flow, so the flows slice's guarantees — one open turn per flow, taken and ended in a
transaction under the tenant row lock, ended `interrupted` at boot — hold for them unchanged. `hub_turns.submission_id`
becomes nullable and `hub_turns` gains `batch_id`, nullable, with a check that exactly one is set. `FlowDriver` takes a
train turn on a flow whenever it is free and its batch is due — just formed, or due for its next proof read on
`TRAIN_POLL_SECONDS` — ahead of any visit's turn, since a batch in progress is the oldest work on its branch. A visit's
turn on a batching flow is a short read of its verdict and runs between the train's.

## The landed event

A batch's landing writes a `landed` fact per member and one `flow-landed` event for the branch's advance. Its `detail`
keeps the flows slice's fields, with `chunk_id` the batch's oldest member, and gains `batch_id` and `chunk_ids`, every
member, so `epic:advanced-deployment` learns in one event that a set of commits landed for a set of chunks. A lone
landing carries a one-element `chunk_ids` and no `batch_id`.

## Events

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `train-unproven` (warning), `train-outpaced`
(warning), `train-advance-refused` (critical), and `forge-capability-fallback` (warning);
`tests/event_log_kind_census.py` and `blizzard-context:/domain/operations.md` gain each.

## Invariants

`tools/invariants.py` gains:

- `hub:one-batch-per-flow` — no flow has two batches in progress.
- `hub:batch-members-open` — every member of a batch in progress is an open submission on the batch's flow, or one
  withdrawn while its batch proved or landed.
- `hub:turn-has-one-subject` — every turn names a submission or a batch, never both and never neither.

`hub:one-turn-per-flow` holds for train turns as for any.
