# The flow's own train

A repository whose `batch_size` is above one lands through a train: its flow gathers the submissions ready on its
branch, merges them onto the base on a branch of its own, proves that one tree with the repository's checks, and
advances the base to exactly that tree. Every step is a short pass the flow driver takes as a turn on the flow, as a
visit's pass is ([record.md](./record.md#the-train-turn)), so a train never holds a process or a long request, and a hub
that restarts mid-train picks it up at the step its record says it reached.

`hub/delivery/trains/` holds it: `readiness.py`, `train.py` (one `step(batch) -> TrainStep` per pass), `split.py`, and
`eject.py`.

## Mechanical first

A train is what keeps an agent from being woken for a branch that merely moved. A member is never brought up to its base
on its own branch, and its change's own checks never gate it: the train merges its revision onto the base and proves the
result, so the base moving under it costs a merge the forge makes, not a rebase an agent makes. Only a conflict against
the base itself goes back to the chunk's graph. The policies slice's mechanical updates — a change brought up to its
base, a fast-forwarding branch merged with its base — still serve a repository that does not batch.

## Readiness

A member's own visit does what belongs to one chunk. When a visit's `land` pass reaches a repository that batches, the
repository's policy does only its preparation: `merge-on-checks` finds or opens the change and writes its
`delivery-pr/<repo>/<n>`, so a reviewer and the board can see it, and both policies gate the head as before. Prepared,
the repository's state is `Queued` — a `RepoState` this slice adds — and the visit writes `train-ready/<repo>` at the
submitted revision, then answers `pending` until the flow's verdict on it is known. Each later pass of the visit reads
that verdict from the flow's record ([record.md](./record.md)): `Landed` once a batch carried it in, the ejection's
verdict once it was ejected, and `Queued` until then. Routing stays the visit's own, through `_route`, exactly as for a
submission landed alone.

A submission is **ready** on a batching flow when its `train-ready` marker is written, its chunk is `delivering`, it is
open on every flow it holds and ready on every one of them that batches — so a chunk reaching two batching repositories
joins neither train until it could join both, which is the chunk atomicity the policies slice keeps, applied at
readiness since two branches never land in one write — and every repository it lands after, in the policies slice's
order, has landed for its chunk.

## Forming

The driver forms a batch on a flow with no batch in progress when its ready submissions number `batch_size`, or the
oldest of them has been ready `batch_wait_seconds`. A batch takes the oldest ready submissions in arrival order, up to
`batch_size`. There is at most one batch in progress on a flow; building speculatively along the line is left for later.

A batch of one whose member is already ahead of the base, holds it, and is green on its own revision is landed as its
policy lands it — a merge for `merge-on-checks`, an advance for `fast-forward` — with no train branch and no second run
of the checks, so a quiet repository lands a lone chunk as promptly as it did before batching.

## Building

The train is a branch, `<train_prefix><flow_id>/<batch_id>`, created at the base's tip with `create_ref`; the batch
records that tip as its base revision. Each member's revision is merged in, in arrival order, with `merge_into`, each
merge recorded against its member as it lands on the train.

A member that will not merge is not, by that alone, in conflict. If it was the first to merge — onto the bare base — it
conflicts with the base, and it is ejected `conflict`. If it merged onto the base but not onto the members ahead of it,
it leaves the batch and keeps its place in the line: once the members ahead of it land it will meet them as part of the
base, and conflict, if it still does, as the first member of a later batch. Every batch's first member therefore either
merges or is ejected, so the line always moves.

## Proving

The batch's revision is the train's tip once every member is merged. The driver reads its checks each pass on the
`TRAIN_POLL_SECONDS` cadence, thirty seconds, with the same reading every policy uses — at least one check, every one
passed or neutral — and the same rule for failures the base shares:

- **Green** — the batch lands ([Landing](#landing)).
- **Still running** — nothing changes; the next pass reads again.
- **A failure the base does not share** — the batch is split ([Splitting](#splitting)).
- **Only failures the base shares** — each is re-run once, as a lone landing's are. Still red, the oldest member is
  ejected `inherited-failure` and the rest return to the head of the line, so one agent repairs the base rather than
  every chunk in the batch.
- **No check at all** after `TRAIN_PROOF_START_SECONDS`, ten minutes — the repository's CI is not running on its train
  branches. The batch is abandoned with a `train-unproven` event naming the repository and the train, and its members
  fall back, for this submission, to landing alone through their policy, written as `train-fallback/<repo>` so their
  next passes take the policy's own path.

## Landing

A proven batch lands with one write: `advance_ref(base, expected=<the batch's base revision>, revision=<its
revision>)`.
The base then holds exactly the tree that was proven, and nothing assembled after the proof reaches it.

- **The base moved** — the advance is refused as stale. The batch is rebuilt on the new tip with the same members and
  proven again; after three rebuilds without landing, its members fall back to landing alone as above, with a
  `train-outpaced` event, since something other than the flow keeps moving the branch.
- **The base is protected** — the advance is refused as protected, which the probe could not see. The batch is abandoned
  with a `train-advance-refused` event, critical, naming the repository; the repository's probe is retaken, which now
  leaves `refs.advance` out, and its members fall back to landing alone.

Once the advance lands, the flow writes each member's `merged/<repo>` marker at the batch's revision, in the member's
visit and epoch through `HubNodeExecutor.record_marker`, which appends its `delivery_repo_landed` row and the flow's
`landed` fact as for any landing. A `merge-on-checks` member's change is read back: GitHub marks a change merged once
its head is reachable from its base; one still open is closed with `close_change`, its comment naming the revision that
carried it in. The train branch is deleted. A hub that dies between the advance and the markers finds, on the batch's
next pass, a base that already holds the batch's revision, and writes what is missing; that the base holds the revision
is the proof the batch landed, whatever else the record says.

## Splitting

A batch that fails is split by arrival order. Its first half is formed as a new batch on the same base — recording the
batch it came from — built, and proven:

- **The first half passes** — it lands, as any batch does. The second half returns to the head of the line, and the next
  batch proves it on the new base, which now holds the first half.
- **The first half fails** — it is split again.
- **A batch of one fails** — its member is the one responsible. It is ejected `failure` with `delivery-findings` naming
  the failing checks on the revision that proved it alone, and every other member of the batch it came from returns to
  the head of the line.

Only a prefix ever lands, so the base takes the members in the order they arrived, and a failure costs at most one proof
per halving.

## Ejecting

An ejection is the flow's verdict on one member: an `ejected` fact on the flow naming the batch, the outcome —
`conflict`, `failure`, or `inherited-failure` — and the findings, which are also written as the member's
`delivery-findings` marker. The member's visit answers that outcome on its next pass and routes along the edge its graph
authored for it, exactly as if it had landed alone and failed; its submission closes. The batch it left moves on without
it, and the flow's record says which chunk was ejected, from which batch, and why.

A lever on a member — stop, restart, complete, migrate — withdraws it as the flows slice withdraws any submission. A
member withdrawn from a batch still building leaves it; one withdrawn while the batch proves or lands stays in it, since
its revision is already in the tree being proven, and the batch lands it if it passes. Its chunk's exit is fenced as the
flows slice fences a stopped chunk's, and the landing is recorded on the flow regardless.
