# The flow's own train

A repository whose `batch_size` is above one lands through a train: its flow gathers the submissions ready on its
branch, merges them onto the base on a branch of its own, proves that one tree with the repository's checks, and
advances the base to exactly that tree. Every step is a short pass the flow driver takes as a turn on the flow, as a
visit's pass is ([record.md](./record.md#the-train-turn)), so a train never holds a process or a long request, and a hub
that restarts mid-train picks it up at the step its record says it reached.

`hub/delivery/trains/` holds it: `readiness.py`, `train.py`, `split.py`, and `eject.py`. A train step is an action under
`bzh:hub-action-purity`, as `land` is — the import allowlist
(`tests/test_layering.py::test_hub_actions_import_only_their_allowlist`) and the frozen-input check
(`tests/test_hub_actions.py`) walk `hub/delivery/trains/` beside `hub/delivery/actions/` — and runs on the `ActionLoop`:
`step(batch, forge, effects)` receives the batch as a frozen value — its members, their revisions, its facts so far, the
flow's repository and settings — the flow's repository's `RepositoryForge`, and an `IEffectSink` through which it hands
over each fact it records; the driver persists each before the step continues, so a forge write and its record are never
further apart than one crash point ([record.md](./record.md#crash-points)). A step that needs more than its batch
carries declares it as an input the driver loads; it never reads the store.

## Mechanical first

A train is what keeps a chunk from paying a run of checks, or an agent, for a branch that merely moved. A member is
never brought up to its base on its own branch: the train merges its revision onto the base and proves the result, so
the base moving under it costs a merge the forge makes. Only a conflict against the base itself goes back to the chunk's
graph. The policies epic's mechanical updates — a change brought up to its base, a fast-forwarding branch merged onto
its base — still serve a repository that does not batch.

A member's own checks need not have finished for it to join, since the train proves it again in combination. One whose
own checks have already failed is not ready: its visit answers as `merge-on-checks` answers a failing change — `failure`
with findings, or `inherited-failure` when the base shares every failure after the one re-run — rather than carrying a
known failure into a batch and costing every other member a split.

## Readiness

A member's own visit does what belongs to one chunk. When a visit's `land` pass reaches a repository that batches, the
repository's policy does only its preparation: `merge-on-checks` finds or opens the change and writes its
`delivery-pr/<repo>/<n>`, so a reviewer and the board can see it, and both policies gate the head as before. Prepared,
the repository's state is `Queued` — a `RepoState` this epic adds — and the visit writes `train-ready/<repo>` at the
submitted revision. Each later pass of the visit reads the flow's verdict on it from the flow's record
([record.md](./record.md)): `Landed` once a batch carried it in, the ejection's verdict once it was ejected, and
`Queued` until then. Routing stays the visit's own, through `_route`, exactly as for a submission landed alone.

`combine.py` folds `Queued` as follows. For the chunk atomicity the policies epic keeps — nothing lands until every
repository the hub lands itself is prepared — `Queued` counts as prepared, as `Ready` does. For the order a project
lands its repositories in, `Queued` is unlanded, so every repository after it waits for its `Landed`. And for the pass's
answer, a visit whose every unlanded repository is `Queued`, or waits behind one, answers `awaiting-queue`
([record.md](./record.md#awaiting-the-queue)), never `pending`, so its node's poll timeout does not turn back a chunk
whose work is sound while its train proves; `awaiting-merge` takes precedence when a parked change is among them.

A submission is **ready** on a batching flow when its `train-ready` marker is written, its chunk is `delivering`, it is
open on every flow it holds, its own checks have not failed, and every repository its chunk reaches that comes before
this one in its project's landing order — the `landing_position` the policies epic fixes in the visit's landing terms at
its first turn, unranked repositories after ranked ones and ties broken by slug — has landed for its chunk. That order
is strict, so a chunk reaching two batching repositories joins their trains one after the other, each once the one
before it has landed, and no repository's readiness waits on another's that waits on it.

## Forming

The driver forms a batch on a flow with no batch in progress and no hold ([Holding](#holding)) when its ready
submissions number `batch_size`, or the oldest of them has been ready `batch_wait_seconds`. A batch takes the oldest
ready submissions in arrival order, up to `batch_size`. There is at most one batch in progress on a flow.

A batch of one whose member's revision already holds the base's tip, and is green on its own revision with the shared
reading, needs no train: the flow lands it with
`advance_ref(base, expected=<the tip>, revision=<the member's
revision>)`, whichever of the two policies it is under, so
the base takes exactly the revision whose checks passed, with no train branch and no second run. A quiet repository
lands a lone chunk as promptly as it did before batching, and the combination is still what lands. A refused advance
sends the member to the train like any other.

## Building

A train is assembled on a staging branch, `<staging_prefix><flow_id>/<batch_id>`, created at the base's tip with
`create_ref`; the batch records that tip as its base revision. Each member's revision is merged in, in arrival order,
with `merge_into`, each merge recorded against its member as it lands on the staging branch. Once every member is in,
the train branch, `<train_prefix><flow_id>/<batch_id>`, is created at the staging branch's tip with one `create_ref`,
and the staging branch is deleted. The repository's checks therefore run once per batch, on its finished tree, rather
than once for every member merged along the way.

A member that will not merge is not, by that alone, in conflict. If it was the first to merge — onto the bare base — it
conflicts with the base, and it is ejected `conflict`. If it merged onto the base but not onto the members ahead of it,
it leaves the batch and keeps its place in the line: once the members ahead of it land it will meet them as part of the
base, and conflict, if it still does, as the first member of a later batch. Every batch's first member therefore either
merges or is ejected, so the line always moves.

## Proving

The batch's revision is the train branch's tip. The driver reads its checks each pass on the `TRAIN_POLL_SECONDS`
cadence, thirty seconds, with the same reading every policy uses — at least one check, every one passed or neutral — and
the same rule for failures the base shares:

- **Green** — the batch lands ([Landing](#landing)).
- **Still running** — nothing changes; the next pass reads again.
- **A failure the base does not share** — the batch is split ([Splitting](#splitting)).
- **Only failures the base shares** — each is re-run once, as a lone landing's are. Still red, the oldest member is
  ejected `inherited-failure`, the rest return to the head of the line, and the flow holds ([Holding](#holding)), so one
  agent repairs the base and no other chunk is ejected for it.
- **No check at all** after `TRAIN_PROOF_START_SECONDS`, ten minutes — the repository's CI is not running on its train
  branches. The batch is abandoned with a `train-unproven` event naming the repository and the train, and its members
  fall back, for this submission, to landing alone through their policy, written as `train-fallback/<repo>` so their
  next passes take the policy's own path.
- **Still not finished** after `TRAIN_PROOF_TIMEOUT`, three hours from the train branch's creation, the hub's default
  poll timeout — a check stuck queued for a runner that never comes, or cancelled by a concurrency group and never run
  again. Every unfinished check is re-run once with `rerun_check` and the bound starts again; still unfinished, the
  batch is abandoned with a `train-proof-timed-out` event naming the checks, and its members return to the head of the
  line to be formed afresh. Both bounds read the clock the driver is given, so tests set them.

## Holding

A flow whose last batch ejected a member `inherited-failure` holds: it forms no batch, and lands no lone member, until
the base's checks are green again. Each `TRAIN_POLL_SECONDS` the driver reads the checks on the base's current tip with
the shared reading; once they pass — on the tip that failed, re-run by someone, or on a newer one that repaired it — the
hold ends and the line moves. The hold is a `train-held` fact and event when it begins and `train-released` when it
ends, and the flow's lane says why it is still. The members waiting meanwhile answer `awaiting-queue`, so the hold costs
them time, not bounces.

## Landing

A proven batch lands with one write: `advance_ref(base, expected=<the batch's base revision>, revision=<its
revision>)`.
The base then holds exactly the tree that was proven, and nothing assembled after the proof reaches it.

- **The base moved** — the advance is refused as stale. The batch is rebuilt on the new tip with the same members and
  proven again; after three rebuilds without landing, its members fall back to landing alone as above, with a
  `train-outpaced` event, since something other than the flow keeps moving the branch.
- **The base is protected** — the advance is refused as protected, which the probe could not see. The batch is abandoned
  with a `train-advance-refused` event, critical, naming the repository; the probe records `advance_refused_at`, which
  leaves `refs.advance` out ([capabilities.md](./capabilities.md)), and its members fall back to landing alone.

Once the advance lands, the flow records the landing itself, keyed by the flow's repository and written in one
transaction: for each member, its `merged/<repo>` marker at the batch's revision on its chunk, its
`delivery_repo_landed` row, and the flow's `landed` fact, with the batch's one `flow-landed` event
([record.md](./record.md#the-landed-event)). It is the flow's write, not a visit's, so it does not pass through a
visit's marker path or its fence: the branch holds the work whatever has become of the visit since, and the record says
so. A `merge-on-checks` member's change is read back: GitHub marks a change merged once its head is reachable from its
base; one still open is closed with `close_change`, its comment naming the revision that carried it in. The train branch
is deleted. A hub that dies between the advance and the record finds, on the batch's next pass, a base that already
holds the batch's revision, and writes what is missing; that the base holds the revision is the proof the batch landed,
whatever else the record says.

A rebuilt, split, or abandoned batch deletes its staging and train branches as it leaves them, and a rebuild recreates
both at the new tip, so no branch outlives the batch that made it.

## Splitting

A batch that fails is split by arrival order. Its first half is formed as a new batch on the same base — recording the
batch it came from — built, and proven:

- **The first half passes** — it lands, as any batch does. The second half returns to the head of the line, and the next
  batch proves it on the new base, which now holds the first half.
- **The first half fails** — it is split again.
- **A batch of one fails** — its failing checks are re-run once, so a flaky check does not cost a chunk its place. Still
  red, its member is the one responsible. It is ejected `failure` with `delivery-findings` naming the failing checks on
  the revision that proved it alone, and every other member of the batch it came from returns to the head of the line.

Only a prefix ever lands, so the base takes the members in the order they arrived, and a failure two members cause
together is charged to the later of them, against a base that already holds the earlier. Each halving costs up to two
proofs — the half that is proven, then the other half proven afresh — so a batch of eight with one bad member lands the
other seven in at most seven proofs.

## Ejecting

An ejection is the flow's verdict on one member: an `ejected` fact on the flow naming the batch, the outcome —
`conflict`, `failure`, or `inherited-failure` — and the findings, which are also written as the member's
`delivery-findings` marker. The member's visit answers that outcome on its next pass and routes along the edge its graph
authored for it, exactly as if it had landed alone and failed; its submission closes on every flow it held. The batch it
left moves on without it, and the flow's record says which chunk was ejected, from which batch, and why.

A chunk that reaches several batching repositories may sit in a batch on one while another ejects it. Its ejection
closes its submission everywhere, so every other batch in progress that carries it is abandoned with a `left-batch` fact
naming the ejection and rebuilt without it, unless that batch's advance has already been made — then its landing stands
and is recorded. A chunk turned back to an agent never has the same work landed behind the agent's back.

A lever on a member — stop, restart, complete — withdraws it as the flows epic withdraws any submission, its outcome
from `withdrawal_for` and deferred to the end of any turn still open on it, and the same rule holds: a member withdrawn
from a batch still building simply leaves it; one withdrawn while its batch proves abandons that batch, which is rebuilt
without it; and one withdrawn after its batch's advance was made has landed, which the flow records as it records any
member. Work a person stopped is never landed by a proof that finished after the stop.
