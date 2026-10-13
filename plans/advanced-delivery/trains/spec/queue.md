# Handing the line to the forge

## The fourth policy

`LandingPolicy` gains `forge-queue`: the hub opens the change, gates it, and hands it to the forge's own queue —
GitHub's merge queue, GitLab's merge train — which orders it, proves it with the queue's own checks, and merges it. The
flow still records every submission, turn, and verdict on the branch, so the operator's view of the line is unchanged;
what it gives up is the ordering, which the forge keeps. Whether the tree the queue merges is the tree it tested is the
forge's promise, not the flow's: GitHub's merge queue lands the merge group it proved, and GitLab's merge train lands
the merged result its pipeline ran on, each by its own rules.

| Policy        | Requires                                                                            | Falls back to, without `queue.join` |
| ------------- | ----------------------------------------------------------------------------------- | ----------------------------------- |
| `forge-queue` | `refs.read`, `compare`, `changes.open`, `changes.read`, `checks.read`, `queue.join` | `merge-on-checks`, unbatched        |

It is the one policy that declares a fallback, and the fallback is applied before any capability is checked:
`requires(repository)` answers the set of the policy that will actually run, so a `forge-queue` repository whose last
probe shows no `queue.join` requires what `merge-on-checks` requires, and the forges slice's check before a pass never
refuses it for the queue it is about to do without. A batch size above one is refused beside `forge-queue`
([capabilities.md](./capabilities.md#the-batching-settings)), so the fallback lands each change alone. Setting the
policy on a repository whose probe does not show `queue.join` is accepted with a warning naming the fallback, rather
than refused, and a visit that finds the queue switched off since lands by the fallback, recording a
`forge-capability-fallback` event (warning) once per visit naming the repository and the capability. Every other missing
capability is still refused when the policy is set and escalated to a person at a visit, as the forges slice does.

## The queue's role

`IForgeQueue` (`hub/forge/landing.py`) is a role of its own beside `IForgeRefs`, `IForgeChanges`, and `IForgeChecks`,
handed out by `ForgeRegistry` for a repository whose probe shows `queue.join`. The frozen `RepositoryForge` an action
receives gains `queue`, set only for such a repository, so an action never holds a queue its repository does not have.
Its operations are async and answer through the forges slice's retry policy:

| Operation                        | Answers                                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `enqueue(number, expected_head)` | The change's `QueueEntry`; `ForgeRejected` when the forge will not take it, with the forge's reason         |
| `queue_entry(number)`            | `QueueEntry(state, position, removal, detail, landed_revision)`, or `None` when the change was never queued |
| `dequeue(number)`                | Whether the forge took the change out                                                                       |

`state` is `queued`, `proving`, `merged`, or `removed`. `removal` says why a change left without merging: `conflict`,
`checks-failed`, `timed-out`, `manual`, `cleared` — taken out by something other than the change, such as the queue
being emptied or rolled back — or `other`, with the forge's own words in `detail`. `landed_revision` is the revision the
forge merged, the same `merged_revision` a change read through `IForgeChanges` carries once merged.

**GitHub.** `enqueue` is the GraphQL `enqueuePullRequest` mutation with `expectedHeadOid`, keyed by the pull request's
node id. `queue_entry` reads the pull request's `mergeQueueEntry` (state, position) and, once it is gone, its newest
`RemovedFromMergeQueueEvent`: `MERGE_CONFLICT` → `conflict`; `CI_FAILURE` → `checks-failed`; `CI_TIMEOUT` → `timed-out`;
`MANUAL` → `manual`; `QUEUE_CLEARED` and `ROLL_BACK` → `cleared`; anything else → `other`. A pull request gone from the
queue and merged — `ALREADY_MERGED`, or merged by the queue — is `merged`, never a removal. `dequeue` is
`dequeuePullRequest`. A queue's checks run on its `merge_group` event, which a repository's CI must listen for, and
without which every change leaves the queue `timed-out`.

**GitLab.** `enqueue` is `POST /projects/:id/merge_trains/merge_requests/:iid` with `sha` and `auto_merge`.
`queue_entry` reads `GET …/merge_trains/merge_requests/:iid`: `idle` → `queued`; `fresh`, `stale`, and `merging` →
`proving`; `merged` and `skip_merged` → `merged`. A change gone from the train unmerged is `removed`, and since GitLab
says why only in a system note, the adapter reads the merge request's newest system note about the train and maps the
phrasings GitLab writes for a failed pipeline, a conflict, and a person's removal; any other is `other`, with the note
in `detail`. `dequeue` cancels the merge request's auto-merge, which takes it off its train. Merge trains are not in
GitLab's free tier, so the routes, the car states, and the system-note phrasings are proven against fixtures recorded
from a Premium instance as well as against the mock.

## Landing through the queue

The `forge-queue` policy (`hub/delivery/actions/landing/forge_queue.py`) is, per repository and pass:

1. Find or open the change, write `delivery-pr/<repo>/<n>`, and gate its head, as `merge-on-checks` does.
2. Read the change's own checks with the shared reading. The forge's queue takes only a change that passes them, so a
   change still running is `Waiting`, and its own failures and inherited failures are verdicts, as they are for
   `merge-on-checks`.
3. Green, it is `Ready`, and its `land` is `enqueue`, after which the visit writes `queued/<repo>/<n>`. The combining
   step's order still holds: a change is not enqueued until every repository its chunk reaches that comes before it in
   its project's landing order has landed.
4. Queued, its state is `Queued`, and each pass reads `queue_entry`. `queued` and `proving` stay `Queued`, which the
   visit answers as `awaiting-queue` ([record.md](./record.md#awaiting-the-queue)); `merged` is `Landed` at
   `landed_revision`; `removed` is a verdict or a person's question:
   - `conflict` → `conflict`; `checks-failed` → `failure` with findings naming the queue's failing checks.
   - `cleared` → a fresh `enqueue` on the next pass, at most twice per visit before it, too, is a `failure` naming the
     reason.
   - `manual` and `timed-out` → an escalation to a person, taken the way the policies slice escalates a parked change a
     person closed unmerged, with `EscalationCause.FORGE_QUEUE_REMOVED` naming the change and the reason, since a person
     who took a change out of the queue, or a queue whose checks never ran, is a person's call and not a defect an agent
     can mend.
   - `other` → `failure`, with the forge's words in the findings.

While its change sits in the forge's queue the visit's turns are short reads, and its submission keeps its place without
holding the flow's next turn from anyone, since the forge, not the flow, decides when it merges. Its wait is not a poll
running long, so the node's poll timeout never turns it back; it is bounded instead by `QUEUE_WAIT_TIMEOUT`, three hours
from the enqueue, past which the visit escalates with `EscalationCause.FORGE_QUEUE_STALLED`, naming the change and its
last place in the queue. The flow's lane shows the forge's position beside its own ([surfaces.md](./surfaces.md)).

A lever that withdraws a queued submission — stop, restart, complete, migrate — also calls `dequeue`, so a stopped
chunk's change does not merge behind the hub's back; a `dequeue` the forge refuses, or a change that merges before it is
dequeued, is recorded as landed on the flow, as a pass already running is under the flows slice.
