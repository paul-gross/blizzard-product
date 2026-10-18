# Submission and verdict

## Submitting

A chunk submits when it stands at a hub node that names an action and its visit has no open submission. Submission is
never an event some caller remembers to send; it is what the driver's reconcile finds true ([below](#the-reconcile)).
The submit resolves the visit's repositories and names the resources it holds
([turns.md](./turns.md#what-a-step-holds)); on a refusal it routes `failure` there and then, as `_refuse` does, and
submits nothing. Otherwise, in one transaction, it creates any flow its resources name that does not yet exist, writes
the submission with the next arrival, a row per flow it holds, and a `submitted` fact to each. A submission is unique on
its visit and generation, `(chunk, node, epoch, generation)`, and a visit is submitted only while it has no open
submission, so a replayed completion, a repeated hub-advance, or two ticks seeing the same chunk never queue it twice;
the generation exists only for the visit whose resources changed under it ([record.md](./record.md#the-tables)).

The chunk is `delivering` from the moment it stands at the node, as it is today: `_latest_movement_enters_hub_node`
decides it, and the submission changes nothing about status.

## The reconcile

Each tick of the flow driver ([turns.md](./turns.md#the-flow-driver)) reconciles every open tenant's lines against where
its chunks stand, before it admits anyone, in two passes:

1. **Every chunk standing at a hub step is in line.** Every chunk that is promoted and not ended, `delivering`, and
   whose newest movement enters a hub node (`NodeShape.is_hub_command_node`, which the actions epic makes
   `executor is HUB`) naming an action, with no open submission for that node at its `latest_epoch`, is submitted,
   oldest entry to the node first. A chunk standing at a hub node that names no action is escalated to a person with the
   actions epic's `hub-action-missing` cause instead, and submits nothing. The actions epic's boot pass escalates the
   same chunks before the driver's first tick, and the escalation is recorded once per `(chunk, node, epoch)`, so
   whichever reaches a chunk first escalates it and the other finds it escalated.
2. **Nothing that has left a step holds a place.** Every open submission with no open turn whose chunk has ended, whose
   chunk's newest movement no longer enters the submission's node, or whose chunk is `delivering` at that node in a
   later epoch is closed with the outcome `withdrawal_for` answers ([record.md](./record.md#the-tables)) — `superseded`
   for a move no lever wrote, the lever's own for one whose write the reconcile reached first — as a `withdrawn` fact on
   each flow it held. A submission with an open turn is left to that turn's end, which closes it the same way; the
   reconcile never frees a branch a pass may still be landing on. A submission whose chunk is `needs_human` at a later
   epoch, or `paused` there — a person may pause a chunk they hold before requeuing it — is left open: it is passed over
   while a person holds the chunk ([escalation](#escalation)), and closed by the first reconcile that finds the chunk
   `delivering` again.

Because the reconcile reads what is true rather than waiting to be told, every path a chunk takes onto or off a hub step
is covered by the same rule: a transition from a worker node, the hub's own transition from one hub node to the next
(`deliver` landing onto `record-findings` in every shipped development graph, written by `_route` through
`record_hub_step_transition` with no runner involved), a migration or restart onto a hub node, a requeue after an
escalation, a resume of a chunk paused before it reached its node, and the first boot after the upgrade. The levers in
[levers.md](./levers.md) still write their withdrawals in their own transactions, so a stopped chunk leaves its lines at
once; the reconcile is the backstop for every move that writes none.

The places that once ran a hub step — `ApplyService._respond`, `_land_migration`, and hub-advance — wake the driver so
the reconcile runs at once rather than on the next interval, and do nothing else.

## The verdict

A turn takes one pass of the visit's action exactly as the slot's run took it, with the same markers and routing, and
the turn's outcome is the choice the pass answered. That outcome is the flow's verdict on the submission whenever it is
an authored choice: `landed`, `conflict`, `failure`, `inherited-failure`, or whichever choice the graph authored. No
outcome name is privileged, here as in `_route`; the flow records the verdict and the chunk routes it.

The verdict reaches the chunk as a fact it already understands: the hub-step transition `_route` writes at `epoch + 1`,
idempotent on `(chunk, from_node, epoch)`. `record_hub_step_transition` (`hub/store/internal/chunk_hub_exec_store.py`)
takes the turn and writes its end, its `turn-ended` facts, and the submission's closure in the same transaction as that
transition, so a flow never shows a verdict its chunk did not take, and a chunk never moves on while its flow still
shows it landing. A verdict closes the submission on every flow it held.

A turn that ends `pending` is no verdict: the submission stays open on its flows ([turns.md](./turns.md#pending)). A
turn whose answer routes nowhere — a choice the graph authored no edge for, or an edge to a node the graph does not
have, which `_route` treats the same way — is a fault in the graph rather than in the chunk's work, and trying again
would only answer the same. `_route` records it as it does today, with the `hub-unroutable-outcome` artifact and the
`hub-node-unroutable-outcome` event, and the actions epic escalates the chunk at once with cause `UNROUTABLE_OUTCOME`,
naming the choice and, for a missing node, the node, where today it re-polls; the turn ends `escalated`.

When `record_hub_step_transition` writes nothing because the exit is a replay or is fenced — the chunk was stopped,
restarted, completed, or moved past the visit while the pass ran — the turn's end and its `turn-ended` facts are still
written in that transaction, and with them the submission's closure, with the outcome `withdrawal_for` answers: a chunk
stopped mid-turn closes `stopped`, not with the pass's answer, since its exit never moved it. A flow therefore never
shows a turn open that has ended, nor a line holding a visit that has left ([levers.md](./levers.md)).

## Escalation

A turn can end by handing its chunk to a person: when the bounce cap is crossed — by an incomplete delivery in `route`
or by a poll timeout in `route_pending_timeout` ([turns.md](./turns.md#where-routing-lives)) — when an answer routes
nowhere, when the visit build finds a capability missing, when the visit's last three turns were cut off
([turns.md](./turns.md#a-turn-that-cannot-end)), and when the action itself escalates a cause no agent can fix. The turn
ends with outcome `escalated`, which is not a verdict.

Today the writes an escalation makes are separate transactions: `record_bounce` and `record_bounce_escalation`
(`hub/store/internal/chunk_escalations_store.py`) each open their own, and `record_bounce` writes unfenced. A turn that
escalates makes its writes in one transaction instead: the bounce where one is counted, the escalation with its hub
lease, the turn's end with outcome `escalated`, and its `turn-ended` facts. The two store methods gain locked variants
that take the transaction the turn's end opens, as `record_hub_step_transition` takes the verdict's, and the bounce is
fenced like the rest. A crash therefore never leaves a chunk escalated with its turn still open, nor a turn ended
`escalated` with no escalation behind it; each write keeps its own replay guard, so a turn ended twice writes nothing
the second time. The submission stays open and keeps its place, passed over while the chunk is `needs_human`
([turns.md](./turns.md#who-is-next)), so the board still shows it in line and the branch moves on without it.

Every escalation that ends a turn opens the next epoch at the node, and the driver relies on it: a requeued chunk is a
new visit only because its escalation raised the epoch. `record_bounce_escalation`
(`hub/store/internal/chunk_escalations_store.py`) writes a hub lease at the epoch it is handed and records the
escalation once per chunk and epoch, and every hub-node escalation goes through its locked variant inside the turn's
ending transaction, handed the visit's epoch plus one: the bounce cap, a poll timeout, an unroutable answer, the actions
epic's `hub-action-missing`, the forges epic's `forge-auth`, `forge-config`, and `forge-capability-missing`, a cause an
action answers itself, and `hub-action-stuck`. None writes an escalation at the visit's own epoch. One that did would
leave the requeued chunk on its old submission, and its next escalation would be refused as already recorded, so the
turn would end `escalated` with nothing behind it and be taken again at once, over and over; the store's own tests
assert each cause takes the epoch above its visit's. A person resolves the escalation as they do today. A stop or a
completion withdraws the submission ([levers.md](./levers.md)). A requeue supersedes the escalation and leaves the chunk
standing at the same node in that later epoch, `delivering` again; the next tick's reconcile closes the old submission
as `superseded` and submits the new visit at the back of every flow it holds, as a restart onto the node does.

## Landing across several repositories

A chunk whose commits resolve to several repositories holds a flow per repository and takes one turn across all of them,
since `land` works every repository in one pass. Its verdict is the one outcome that run reaches, written to every flow
it held.

It is landed only when every repository is: `_route` still compares the chunk's commits against `LandedRepos.names` and
records a bounce with its `unlanded` list when a non-terminal route leaves one behind. A repository that landed before
another was refused keeps its `merged/<repo>` marker, its `delivery_repo_landed` row, and its flow's `landed` fact; a
later visit skips it, as the `land` action's marker skip does. A later visit is a new submission, at the back of every
flow it holds, and holds only the flows of repositories still unlanded.

A chunk keeps holding the flow of a repository that landed earlier in the same visit until that visit's verdict. Its
turns are one pass each, so the branch is held a pass at a time and never through a wait.

Each of those repositories lands against its own record — its own base branch, owner, forge address, and credential — so
the flows one chunk holds may be on different branches as well as different repositories; nothing refuses a chunk whose
repositories differ. Which policy each repository lands by, and in what order a chunk's repositories land, are the
`policies` epic's.
