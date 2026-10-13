# Submission and verdict

## Submitting

A chunk submits when it arrives at a hub command node: a transition, migration, or restart onto it. The submit resolves
the visit's repositories ([turns.md](./turns.md#what-a-step-holds)); on a refusal it routes `failure` there and then, as
`_refuse` does, and submits nothing. Otherwise, in one transaction, it creates any flow its resources name that does not
yet exist, writes the submission with the next arrival, a row per flow it holds, and a `submitted` fact to each, then
wakes the flow driver. A submission is idempotent on its visit, so a replayed completion or a repeated hub-advance never
queues a chunk twice.

The chunk is `delivering` from that moment, as it is today: `_latest_movement_enters_hub_node` decides it, and the
submission changes nothing about status.

## The verdict

A turn takes one pass of the visit's action exactly as the slot's run took it, with the same markers and routing, and
the turn's outcome is the choice the pass answered. That outcome is the flow's verdict on the submission whenever it is
anything but `pending`: `landed`, `conflict`, `failure`, `inherited-failure`, or whichever choice the graph authored. No
outcome name is privileged, here as in `_route`; the flow records the verdict and the chunk routes it.

The verdict reaches the chunk as a fact it already understands: the hub-step transition `_route` writes at `epoch + 1`,
idempotent on `(chunk, from_node, epoch)`. `record_hub_step_transition` (`hub/store/internal/chunk_hub_exec_store.py`)
takes the turn and writes its end and `turn-ended` facts in the same transaction as that transition, so a flow never
shows a verdict its chunk did not take, and a chunk never moves on while its flow still shows it landing. A verdict
closes the submission on every flow it held.

A turn that ends `pending` is no verdict: the submission stays open on its flows ([turns.md](./turns.md#pending)). A
turn that routes nowhere — an unroutable outcome, which `_route` records as `hub-unroutable-outcome`, or an edge to a
node the graph does not have, which `_route` treats the same way — ends with that outcome and leaves the submission
open, so the chunk is tried again at its next poll as it is today, without holding its branch between tries.

`_route` can also end a visit without a transition. When the bounce cap is crossed — by an incomplete delivery in
`_route` or by a poll timeout in `_route_pending_timeout` — it records the escalation and writes no transition; the turn
ends with outcome `escalated`, which is its verdict, and the submission closes on every flow it held, since the chunk
has left delivery for a person's hands. And when `record_hub_step_transition` writes nothing because the exit is a
replay or is fenced — the chunk was stopped, restarted, or moved past the visit while the pass ran — the turn's end and
its `turn-ended` facts are still written in that transaction, so a flow never shows a turn open that has ended
([levers.md](./levers.md)).

## Landing across several repositories

A chunk whose commits resolve to several repositories holds a flow per repository and takes one turn across all of them,
since `land` works every repository in one pass. Its verdict is the one outcome that run reaches, written to every flow
it held.

It is landed only when every repository is: `_route` still compares the chunk's commits against `LandedRepos.names` and
records a bounce with its `unlanded` list when a non-terminal route leaves one behind. A repository that landed before
another was refused keeps its `merged/<repo>` marker, its `delivery_repo_landed` row, and its flow's `landed` fact; a
later visit skips it, as the `land` action's marker skip does (the scripts' `LandRun.pending()` before the actions
slice). A later visit is a new submission, at the back of every flow it holds, and holds only the flows of repositories
still unlanded.

Each of those repositories lands against its own record — its own base branch, owner, forge, and credential — so the
flows one chunk holds may be on different branches as well as different repositories; nothing refuses a chunk whose
repositories differ. Which policy each repository lands by, and in what order a chunk's repositories land, are the
`policies` slice's.
