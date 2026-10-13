# Handing the line to the forge

## The fourth policy

`LandingPolicy` gains `forge-queue`: the hub opens the change, gates it, and hands it to the forge's own queue —
GitHub's merge queue, GitLab's merge train — which orders it, proves it with the queue's own checks, and merges it. The
flow still records every submission, turn, and verdict on the branch, so the operator's view of the line is unchanged;
what it gives up is the ordering, which the forge keeps.

| Policy        | Requires                                                                            | Falls back to, without `queue.join`                              |
| ------------- | ----------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `forge-queue` | `refs.read`, `compare`, `changes.open`, `changes.read`, `checks.read`, `queue.join` | `merge-on-checks`, batched as the repository's `batch_size` says |

It is the one policy that declares a fallback. Setting it on a repository whose probe does not show `queue.join` is
accepted with a warning naming the fallback, rather than refused, and a visit that finds the queue switched off since
lands by the fallback, recording a `forge-capability-fallback` event (warning) once per visit naming the repository and
the capability. Every other missing capability is still refused, as the policies slice refuses it.

## The seam's queue

`IForgeLanding` gains the queue's operations, all behind `queue.join`:

| Operation                        | Answers                                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `enqueue(number, expected_head)` | The change's `QueueEntry`; `ForgeRejected` when the forge will not take it, with the forge's reason         |
| `queue_entry(number)`            | `QueueEntry(state, position, removal, detail, landed_revision)`, or `None` when the change was never queued |
| `dequeue(number)`                | Whether the forge took the change out                                                                       |

`state` is `queued`, `proving`, `merged`, or `removed`. `removal` says why a change left without merging: `conflict`,
`checks-failed`, `manual`, `cleared` — taken out by something other than the change, such as the queue being emptied or
rolled back — or `other`, with the forge's own words in `detail`.

**GitHub.** `enqueue` is `PUT /repos/{o}/{r}/pulls/{n}/merge-async` with `merge_action: merge_queue` and `sha` as the
expected head, answered 202 and read back through `GET …/merge-async/{id}`. `queue_entry` reads the pull request's
`mergeQueueEntry` (state, position) and, once it is gone, its newest `RemovedFromMergeQueueEvent` over GraphQL:
`MERGE_CONFLICT` → `conflict`; `CI_FAILURE` and `CI_TIMEOUT` → `checks-failed`; `MANUAL` → `manual`; `QUEUE_CLEARED` and
`ROLL_BACK` → `cleared`; anything else → `other`. `dequeue` is `dequeuePullRequest`.

**GitLab.** `enqueue` is `POST /projects/:id/merge_trains/merge_requests/:iid` with `sha` and `auto_merge`.
`queue_entry` reads `GET …/merge_trains/merge_requests/:iid`: `idle` → `queued`; `fresh` and `stale` → `proving`;
`merged` and `skip_merged` → `merged`. A change gone from the train unmerged is `removed`, and since GitLab says why
only in a system note, the adapter reads the merge request's newest system note about the train and maps the phrasings
GitLab writes for a failed pipeline, a conflict, and a person's removal; any other is `other`, with the note in
`detail`. `dequeue` is `POST …/merge_requests/:iid/cancel_auto_merge`.

## Landing through the queue

The `forge-queue` policy (`hub/delivery/actions/landing/forge_queue.py`) is, per repository and pass:

1. Find or open the change, write `delivery-pr/<repo>/<n>`, and gate its head, as `merge-on-checks` does.
2. Read the change's own checks with the shared reading. The forge's queue takes only a change that passes them, so a
   change still running is `Waiting`, and its own failures and inherited failures are verdicts, as they are for
   `merge-on-checks`.
3. Green, it is `Ready`, and its `land` is `enqueue`, after which the visit writes `queued/<repo>/<n>`. The combining
   step's order still holds: a change whose predecessors have not landed is not enqueued yet.
4. Queued, each pass reads `queue_entry`. `queued` and `proving` are `Waiting`; `merged` is `Landed` at the forge's
   merge revision; `removed` is a verdict — `conflict` for a conflict, `failure` with findings for failed checks or a
   person's removal, and for `cleared` a fresh `enqueue` on the next pass, at most twice per visit before it, too, is a
   `failure` naming the reason.

While its change sits in the forge's queue the visit's turns are short reads, and its submission keeps its place without
holding the flow's next turn from anyone, since the forge, not the flow, decides when it merges. The flow's lane shows
the forge's position beside its own ([surfaces.md](./surfaces.md)).

A lever that withdraws a queued submission — stop, restart, complete, migrate — also calls `dequeue`, so a stopped
chunk's change does not merge behind the hub's back; a `dequeue` the forge refuses, or a change that merges before it is
dequeued, is recorded as landed on the flow, as a pass already running is under the flows slice.
