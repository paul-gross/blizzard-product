# Submission and verdict

## Submitting

A chunk submits when it stands at a hub node that names an action and its visit has no submission yet. Submission is
never an event some caller remembers to send; it is what the driver's reconcile finds true ([below](#the-reconcile)).
The submit resolves the visit's repositories ([turns.md](./turns.md#what-a-step-holds)); on a refusal it routes
`failure` there and then, as `_refuse` does, and submits nothing. Otherwise, in one transaction, it creates any flow its
resources name that does not yet exist, writes the submission with the next arrival, a row per flow it holds, and a
`submitted` fact to each. A submission is unique on its visit, `(chunk, node, epoch)`, so a replayed completion, a
repeated hub-advance, or two ticks seeing the same chunk never queue it twice.

The chunk is `delivering` from the moment it stands at the node, as it is today: `_latest_movement_enters_hub_node`
decides it, and the submission changes nothing about status.

## The reconcile

Each tick of the flow driver ([turns.md](./turns.md#the-flow-driver)) reconciles every open tenant's lines against where
its chunks stand, before it admits anyone, in two passes:

1. **Every chunk standing at a hub step is in line.** Every chunk that is promoted and not ended, `delivering`, and
   whose newest movement enters a hub node (`NodeShape.is_hub_command_node`, which the actions slice makes
   `executor is HUB`) naming an action, with no submission for that node at its `latest_epoch`, is submitted, oldest
   entry to the node first. A chunk standing at a hub node that names no action is escalated to a person with the
   actions slice's `hub-action-missing` cause instead, and submits nothing.
2. **Nothing that has left a step holds a place.** Every open submission whose chunk has ended, whose chunk's newest
   movement no longer enters the submission's node, or whose chunk is `delivering` at that node in a later epoch is
   closed with outcome `superseded`, as a `withdrawn` fact on each flow it held. A submission whose chunk is
   `needs_human` at a later epoch is left open: it is passed over while a person holds the chunk
   ([escalation](#escalation)).

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
turn whose answer routes nowhere — a choice the graph authored no edge for, which `_route` records as
`hub-unroutable-outcome`, or an edge to a node the graph does not have, which `_route` treats the same way — is a fault
in the graph rather than in the chunk's work, and trying again would only answer the same. The chunk is escalated to a
person at once with the actions slice's `UNROUTABLE_OUTCOME` cause, naming the choice and, for a missing node, the node;
the turn ends `escalated`.

When `record_hub_step_transition` writes nothing because the exit is a replay or is fenced — the chunk was stopped,
restarted, or moved past the visit while the pass ran — the turn's end and its `turn-ended` facts are still written in
that transaction, so a flow never shows a turn open that has ended ([levers.md](./levers.md)).

## Escalation

A turn can end by handing its chunk to a person: when the bounce cap is crossed — by an incomplete delivery in `_route`
or by a poll timeout in `_route_pending_timeout` — when an answer routes nowhere, and when the action itself escalates a
cause no agent can fix. The turn ends with outcome `escalated`, which is not a verdict. The submission stays open and
keeps its place, passed over while the chunk is `needs_human` ([turns.md](./turns.md#who-is-next)), so the board still
shows it in line and the branch moves on without it.

Every escalation of a hub-node visit opens the next epoch at the node, as `record_bounce_escalation`'s hub lease does
today (`hub/store/internal/chunk_escalations_store.py`), so a requeued chunk is always a new visit. A person resolves
the escalation as they do today. A stop or a completion withdraws the submission ([levers.md](./levers.md)). A requeue
supersedes the escalation and leaves the chunk standing at the same node in that later epoch, `delivering` again; the
next tick's reconcile closes the old submission as `superseded` and submits the new visit at the back of every flow it
holds, as a restart onto the node does.

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
`policies` slice's.
