# Operator levers

What each lever means is unchanged and owned by `CHUNK_VERB_LEGALITY` (`hub/domain/chunk/model.py`) and
`blizzard-context:/domain/work/`. This slice changes what each does to a chunk's place in its flows, and stops any of
them from freeing a branch while a pass still runs on it.

| Lever    | On a submission waiting its turn                                                                                  | On a submission mid-turn                                                                                                                          |
| -------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| pause    | Refused at `delivering`, as today (`ChunkNotPausable`). A chunk paused before it reached the node is passed over. | Refused, as today.                                                                                                                                |
| stop     | Withdrawn: a `withdrawn` fact, outcome `stopped`, on every flow it held, in `record_stop_locked`'s transaction.   | The pass runs to its end and its verdict is recorded on the flow; the chunk's exit is fenced as terminal, as today. Withdrawn when the turn ends. |
| restart  | Withdrawn, outcome `restarted`. A restart onto the same hub node is a new visit and submits again, at the back.   | As stop: the pass runs to its end, its exit is fenced as a stale epoch, and the old visit is withdrawn when the turn ends.                        |
| complete | Withdrawn, outcome `completed`.                                                                                   | As stop.                                                                                                                                          |
| migrate  | Withdrawn, outcome `migrated`; a migration onto a hub node submits the new visit.                                 | As stop.                                                                                                                                          |
| detach   | Unchanged. The submission stays: the flow lands the chunk without its runner.                                     | Unchanged.                                                                                                                                        |

A pass already running is never cancelled by a lever. A merge it makes has happened whatever the chunk's status, so its
markers are written at `AT_OR_ABOVE` as today and its `landed` fact is recorded on the flow; only the chunk's exit is
fenced. The flow stays busy until the pass ends, so no other landing starts on a branch a stopped chunk may still be
merging into.

`record_stop_locked` and `record_completion_locked` (`hub/store/internal/chunk_lifecycle_store.py`) stop releasing the
hub-execution slot, which no longer exists, and write the withdrawal instead. They release it today while a pass may
still be running, so a stop frees the branch mid-pass; the deferral below closes that. Restart and migration release
nothing today, so their withdrawals are new: the restart and migration writes gain them in their own transactions. A
withdrawal against a submission with an open turn is deferred to the turn's end: the turn's ending transaction sees the
chunk is stopped, restarted, completed, or migrated past the visit and writes the `withdrawn` fact beside `turn-ended`.

Paused chunks at a hub node, which hub-advance and `ApplyService` would run today, take no turn: eligibility requires
`delivering` ([turns.md](./turns.md#who-is-next)).
