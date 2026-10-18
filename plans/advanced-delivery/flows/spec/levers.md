# Operator levers

What each lever means is unchanged and owned by `CHUNK_VERB_LEGALITY` (`hub/domain/chunk/model.py`) and
`blizzard-context:/domain/work/`. This epic changes what each does to a chunk's place in its flows, and stops any of
them from freeing a branch while a pass still runs on it. A lever that moves a chunk onto a hub step never submits it
itself: the driver's reconcile does, on the tick the lever wakes ([verdict.md](./verdict.md#the-reconcile)).

| Lever    | On a submission waiting its turn                                                                                                                                                                                             | On a submission mid-turn                                                                                                                                         |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| pause    | Refused at `delivering`, as today (`ChunkNotPausable`). A chunk paused before it reached the node is not submitted until it is resumed; one a person pauses while it is `needs_human` keeps its old submission, passed over. | Refused, as today.                                                                                                                                               |
| resume   | A chunk resumed at a hub node is `delivering` again and is submitted by the next reconcile, at the back.                                                                                                                     | Not reachable: a chunk mid-turn is never paused.                                                                                                                 |
| stop     | Withdrawn: a `withdrawn` fact, outcome `stopped`, on every flow it held, in `record_stop_locked`'s transaction.                                                                                                              | The pass runs to its end; a merge it made is recorded on the flow, and its exit is fenced as terminal, as today. The turn's end closes the submission `stopped`. |
| restart  | Withdrawn, outcome `restarted`. A restart onto the same hub node is a new visit, submitted by the next reconcile at the back.                                                                                                | As stop: the pass runs to its end, its exit is fenced as a stale epoch, and the turn's end closes the old visit `restarted`.                                     |
| complete | Withdrawn, outcome `completed`.                                                                                                                                                                                              | As stop, closing `completed`.                                                                                                                                    |
| requeue  | A chunk escalated while in line keeps its submission until the requeue; the requeued visit is new, and the next reconcile supersedes the old and submits it at the back.                                                     | Not reachable: a turn that escalates has ended.                                                                                                                  |
| detach   | Unchanged. The submission stays: the flow lands the chunk without its runner.                                                                                                                                                | Unchanged.                                                                                                                                                       |

A pass already running is never cancelled by a lever. A merge it makes has happened whatever the chunk's status, so it
is recorded whatever the chunk's status. The chunk's fence refuses its terminal and stale-epoch writes — `fence()`
(`hub/store/internal/chunk_rows.py`) refuses a stopped or completed chunk before it looks at the epoch, so a plain
marker write would be dropped — and the actions epic's `LandingRecord` effect is the one write admitted past it: the
merge's `merged/<repo>` marker and its `delivery_repo_landed` row, to which this epic adds the flow's `landed` fact and
`flow-landed` event in the same transaction ([record.md](./record.md#what-the-record-says)). The sink then answers the
pass `fenced`, and `land` touches no further repository: a stopped chunk's landing stops at the repository it was
merging, and whatever it had not reached stays unlanded. Only the chunk's exit is fenced. The flow stays busy until the
pass ends, so no other landing starts on a branch a stopped chunk may still be merging into.

`record_stop_locked` and `record_completion_locked` (`hub/store/internal/chunk_lifecycle_store.py`) stop releasing the
hub-execution slot, which no longer exists, and write the withdrawal instead. They release it today while a pass may
still be running, so a stop frees the branch mid-pass; the deferral below closes that. A restart releases nothing today,
so its withdrawal is new: the restart write gains it in its own transaction. A withdrawal against a submission with an
open turn is deferred to the turn's end, and the lever writes nothing to the flow: the turn's ending transaction asks
`withdrawal_for` ([record.md](./record.md#the-tables)), finds the chunk stopped, restarted, or completed past the visit,
and writes the closure and its `withdrawn` fact beside `turn-ended`. Should that end never be written — the hub stopped,
or the write failed — the in-tick sweep or boot recovery that ends the turn `interrupted` asks the same question in the
same transaction, so a deferred withdrawal is recorded with the lever's outcome however the turn ends. A move that
writes no withdrawal of its own is caught by the reconcile, which closes the submission as `superseded`.

A cross-graph migration needs no row of its own. A migration fact is written in two places: on a worker's completion
(`ApplyService`, `hub/domain/execution/apply.py`), which moves a chunk onto a hub node, where the reconcile submits it,
but never off one while it holds a place; and by a restart that names another graph
(`hub/domain/operations/restart.py`), whose restart withdrawal already covers it.

A chunk paused at a hub node, which hub-advance and `ApplyService` would run once today while it is paused, now takes no
turn until it is resumed: eligibility requires `delivering` ([turns.md](./turns.md#who-is-next)). This is the one
landing that changes under this epic, and only in when it starts.
