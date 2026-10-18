# The train's record

## The tables

Every table here is tenant-owned and written through `ScopedConnections`, beside the flows epic's tables in
`hub/store/schema.py`. A batch is made by the work that reaches it, so, like a flow, it has no revision and no change
log.

```text
flow_batches
  batch_id          String       NOT NULL   fbt_<ulid>
  tenant_id         String       NOT NULL
  flow_id           String       NOT NULL   FK flows
  split_from        String       NULL       the batch this one was split from
  base_revision     String       NOT NULL   the base's tip the train was built on; changes on a rebuild
  staging_ref       String       NOT NULL   <staging_prefix><flow_id>/<batch_id>
  train_ref         String       NOT NULL   <train_prefix><flow_id>/<batch_id>
  revision          String       NULL       the train's tip once every member is merged
  formed_at         UtcDateTime  NOT NULL
  PRIMARY KEY (batch_id)

flow_batch_members
  batch_id          String       NOT NULL   FK flow_batches
  submission_id     String       NOT NULL   FK flow_submissions
  tenant_id         String       NOT NULL
  position          Integer      NOT NULL   arrival order within the batch
  merged_revision   String       NULL       the staging branch's tip once this member was merged in
  PRIMARY KEY (batch_id, submission_id)
```

`IdPrefix` (`foundation/ids.py`) gains `FLOW_BATCH = "fbt"`. Domain types join `hub/domain/delivery/flows.py` — `Batch`,
`BatchMember`, and `BatchState`, derived — and the store `flow_store.py`, behind the same ports. The driver reads a
flow's batch in progress and its members by `flow_batches (tenant_id, flow_id)` and
`flow_batch_members (tenant_id, submission_id)`, both indexed.

`flow_facts` gains a nullable `batch_id`, and its `submission_id` and `chunk_id` become nullable, since a batch's own
facts and a hold belong to the flow rather than to one submission; a check keeps every fact naming a submission, a
batch, or — for a hold — neither but its flow. Its `kind` gains `batch-formed`, `batch-built`, `batch-landed`,
`batch-split`, `batch-abandoned`, `left-batch`, `ejected`, `train-held`, and `train-released`; `outcome` carries the
ejection's verdict or the reason a batch was abandoned — `unproven`, `outpaced`, `advance-refused`, `proof-timed-out`,
or `member-left`, when a member's ejection elsewhere or its withdrawal took it out of a batch already proving. A batch's
state — building, proving, landed, split, abandoned — is derived from its facts and its member rows
(`bzh:facts-not-status`), never stored, and so is a flow's hold: the newest `train-held` or `train-released`. A member's
verdict is the newest `landed` or `ejected` fact naming its submission.

## The train turn

A train's passes are turns on its flow, so the flows epic's guarantees — one open turn per flow, taken and ended in a
transaction under the tenant row lock, ended `interrupted` at boot and by the driver's own sweep of turns it is not
running — hold for them unchanged. `hub_turns.submission_id` becomes nullable and `hub_turns` gains `batch_id`,
nullable, with a check that exactly one is set. `FlowDriver` takes a train turn on a flow whenever it is free and its
batch is due — just formed, or due for its next proof read on `TRAIN_POLL_SECONDS` — or, while the flow holds, its next
read of the base's checks; a train turn goes ahead of any visit's turn, since a batch in progress is the oldest work on
its branch. A visit's turn on a batching flow is a short read of its verdict and runs between the train's.

The train's step runs on the `ActionLoop` as every action does, receiving the frozen batch and handing its facts to an
`IEffectSink`; the driver writes each fact through `to_thread` before the step continues, and holds no store transaction
across the step. A train turn ends with one of the flows epic's turn outcomes — `pending` while its batch proves,
`interrupted` when cut short — and a member's ejection is the verdict that closes its submission in
`flow_submission_closures`, as any verdict does.

## Awaiting the queue

`awaiting-queue` is reserved beside `pending` and `awaiting-merge` (`HUB_AWAITING_QUEUE_CHOICE` in
`hub/domain/graph/model.py`): never authored, refused as a choice at mint by `validation.py` as they are, and never a
verdict. A pass answers it when its chunk's unlanded repositories are `Queued` on a batching flow or in a forge's queue,
or wait behind one ([batches.md](./batches.md#readiness)). The pass ends its turn with that outcome; its submission
keeps its place on its flows and is ineligible until its next poll is due, after the node's `poll_interval`, as after
`pending`.

`hub_node_poll.kind`, which the policies epic added, gains `awaiting-queue`, written by the driver's `_record_wait` as
`awaiting-merge` is, and `awaiting-queue` joins the actions epic's `RESERVED_WAITS`. Like `awaiting-merge`, it ends a
run of `pending` rows, so the node's `poll_timeout` never turns back a chunk for the time a train or a forge's queue
takes to prove it. Its bound is the train's or the queue's own: `TRAIN_PROOF_TIMEOUT` on a batch
([batches.md](./batches.md#proving)) and `QUEUE_WAIT_TIMEOUT` on a change in a forge's queue
([queue.md](./queue.md#landing-through-the-queue)).

## The landed event

A batch's landing is the flow's own write, made in one transaction keyed by the flow's repository:
`flow_store.record_batch_landed` appends, for each member, its `merged/<repo>` marker on its chunk at the batch's
revision, its `delivery_repo_landed` row, and the flow's `landed` fact, and then the batch's `batch-landed` fact and one
`flow-landed` event for the branch's advance. It writes those markers without a visit's fence, since the branch holds
the work whatever has since become of the member's visit. The flows epic's own per-landing `flow-landed` is not emitted
for a batch's members; the batch's one event stands for them.

The event's `detail` keeps the flows epic's fields, describing the batch's oldest member, and gains `batch_id` and
`members`, each a `chunk_id` with its `project_id`, so whatever follows a landing learns in one event that a set of
commits landed for a set of chunks. A lone landing carries a one-element `members` and no `batch_id`.

## Events and escalation causes

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `train-unproven` (warning), `train-outpaced`
(warning), `train-advance-refused` (critical), `train-proof-timed-out` (warning), `train-held` (warning),
`train-released` (info), and `forge-capability-fallback` (warning); `tests/event_log_kind_census.py` and
`blizzard-context:/domain/operations.md` gain each.

`EscalationCause` (`foundation/escalation_causes.py`) gains `FORGE_QUEUE_REMOVED`, for a change a person took out of a
forge's queue or whose queue's checks never ran, and `FORGE_QUEUE_STALLED`, for a change still in a forge's queue past
`QUEUE_WAIT_TIMEOUT`.

## Crash points

Each window between a forge write and its record, and between a decision and the facts that carry it out, is a named
point in the registry (`bzh:crash-point-registry`), each reached by a scenario of the kill-9 sweep
(`tests/crash/test_kill9_sweep.py`) through the test-fenced crash-point mechanism:

| Point                               | The window                                                                      | What the next pass finds                                                                                                    |
| ----------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `train.after-ref.before-record`     | A `create_ref`, `merge_into`, or `delete_ref` answered; its fact is not durable | The branch as the forge holds it: a ref already at its revision, or a merge with nothing left to merge, is recorded as done |
| `train.after-advance.before-landed` | `advance_ref` accepted; the batch's landing record is not durable               | A base that holds the batch's revision, and writes the whole landing record                                                 |
| `train.after-enqueue.before-queued` | The forge took a change into its queue; `queued/<repo>/<n>` is not durable      | `queue_entry` answers the change queued, and the marker is written without a second `enqueue`                               |
| `train.after-split.before-eject`    | `batch-split` is durable; the halves and the ejection they imply are not        | The split re-derived from its fact, forming the half or ejecting the member it names                                        |

## Invariants

`tools/invariants.py` gains:

- `hub:one-batch-per-flow` — no flow has two batches in progress.
- `hub:batch-members-open` — every member of a batch in progress is an open submission on the batch's flow, and no chunk
  withdrawn or ejected is a member of a batch still proving.
- `hub:turn-has-one-subject` — every turn names a submission or a batch, never both and never neither.
- `hub:flow-fact-has-subject` — every flow fact names a submission, a batch, or is a hold of its flow.

`hub:one-turn-per-flow` holds for train turns as for any.
