# Minting an errand

## The sweep

The hub looks over its cadences in a seventh sweep, `cadence`, every 60 seconds and once as it boots, registered beside
the other six with its own interval and its `[sweeps.cadence] mode` key, so the hub's configuration contract gains the
key. It is patterned on the close drain: it reads the corpus of every open tenant once through the hub-scoped reads,
then does each tenant's work through that tenant's own services (`bzh:sweep-acts-in-row-tenant`), each tenant inside its
own catch (`bzh:pass-isolates-tenants`), and it joins the poisoned-tenant fixture every such pass is proven by. It
serves one tenant on request as well, so `POST /api/sweeps/cadence/passes` and its admin twin step it. It is a pass that
acts on work as it comes due, which `bzh:probe-gated-pass` leaves without a probe or a floor: time passing is its only
signal.

A pass, for each tenant:

1. **Reads the configuration.** The tenant's cadences not retired, with their chores and listed scopes; the routines
   they name, in bulk; which of those routines' graphs resolve; and the scopes they name with each routine's linked
   scopes. These say what the errands are and which can run.
2. **Reads the runs.** One read, the snapshot the decision rests on (`bzh:pass-reads-one-snapshot`): each errand's last
   run and its chunk's status ([due.md](./due.md) §The last run). Anything the configuration read saw that has moved
   since is caught by the mint's own refusals, and anything the run read saw that has moved since is caught under the
   mint's lock.
3. **Judges every errand** ([due.md](./due.md) §The due rule), skipping any the cadence cannot run
   ([surfaces.md](./surfaces.md) §Needs attention).
4. **Mints each due errand** in a unit of work of its own ([§The mint's unit of work](#the-mints-unit-of-work)), so one
   errand's refusal or conflict never holds back another.

A pass is bounded: it mints at most 50 errands, shared fairly across tenants as the close drain shares its budget,
starting from a different tenant each pass. An errand left over is still due on the next pass a minute later, and is
minted once there. A fleet switched off for a week comes back to a few passes of minting, never to a pile of runs.

## What the mint reuses

An errand is minted as a hand-started run is, and nothing goes around the garden's rules: the same refusals in the same
order — a retired routine, a graph that no longer resolves, a scope the routine does not sweep, a retired scope — the
same settling of a `delta` with no baseline down to `full`, recorded as downgraded, the same charge with the chore's
note as its *This run* section, the same title, and the work minted in the routine's project, which is the cadence's.
The garden's run planning, which is pure, is the step both paths share. A hand-started run keeps its own path and is
minted exactly as it is today.

The delta baseline is still the routine's own over that scope, kept by routine and scope rather than by run. A monthly
full run from one chore therefore sets the baseline the next weekly delta from another chore reads, as the plan intends.

## The mint's unit of work

Each mint is an exactly-one-wins decision (`bzh:write-reestablishes-read`), and promoting is a decision about the queue,
so the cadence mint is a domain unit of work of the cadence package, run through the hub's transaction seam
(`bzh:unit-of-work`), and not a store write with a promote bolted on. The pass hands it the errand, the run it judged to
be the errand's last (or that it judged there was none), and the planned run. Inside one transaction, in this order:

1. **The errand's lock.** A keyed lock on the errand, its chore and scope.
2. **The ranking locks**, for a promoting chore only: the locks the queue's own promote takes on its lists, in the order
   it takes them.
3. **The re-read.** The errand's last run, read again. If it is not the run the pass judged, or it is unfinished, the
   mint writes nothing: another pass, or a request on the route, got there first. A lost race is not an error; the pass
   logs it at debug and moves on.
4. **The run.** The work item, its chunk, and the run's identity with the errand's mark
   ([§The errand's mark](#the-errands-mark)).
5. **The landing**, for a promoting chore: the run's place at the tail of the ready list, decided under the ranking
   locks as the queue's own promote decides it, and the promote fact and position written beside the run.

The lock order — errand, then ranking — is recorded with the hub's other lock orders and proven on Postgres by
`blizzard:postgres-uow-concurrency`. A run minted promoted never rests `not_ready`, even for an instant a runner could
see, and its position can never collide with an operator's promote or reorder landing at the same moment.

## The errand's mark

The run's identity record gains two nullable columns, `cadence_id` and `chore_id`, and an index on its tenant, chore,
and scope for the newest-per-errand read. They are written with the run's identity in the mint's transaction, so no run
exists without its mark and no mark without its run. A hand-started run leaves both null. A run's identity as the hub
reads it carries the mark, so a run's view can name the cadence and chore that minted it ([surfaces.md](./surfaces.md)
§Runs). The columns arrive in an additive hub revision of their own, landed with the sweep
([carry-over.md](./carry-over.md)).

## The author

A work item's author is a person or a fleet node today. A cadence's run is neither, so the author gains a third kind,
`cadence`, carrying the cadence's id and the chore's id. The author is already stored as a document, so the store needs
no change. Every view that answers an author's kind declares it open (`bzh:open-response-vocabulary`); where one is
closed today, it is made open in the change that adds the kind.

## Promoting

A chore that promotes admits its run to the ready queue in the mint's own unit of work. A chore with `promote: false`
mints its run exactly as a hand-started run is minted, resting behind the gate until someone promotes it.

This is the one admission to the queue beside an operator's. The domain's statement of the promote gate, in the chunk's
statuses and in the routine's run, says so: a run rests behind the promote gate unless the cadence chore that minted it
promotes. An accepted garden proposal still rests behind the gate, and is unchanged.

## Refused, conflicted, faulted

An errand the configuration read showed runnable may stop being so before its mint, because its routine was retired or
its scope unlinked a moment earlier. The shared run planning then refuses it as it would refuse an operator. The pass
logs the refusal, mints nothing, and the next read shows the attention it needs. An ingest conflict, which the
hand-started route answers `409`, is handled the same way, for that errand alone. Any other exception from one errand is
caught and logged, and the pass carries on; a tenant whose reads fail outright is that tenant's fault, the other tenants
untouched. None of these is latched or retried with backoff: the next pass, a minute on, judges the errand afresh
(`bzh:lane-contract`).

## After the commit

Each mint, once committed, publishes what the hand-started route publishes: the chunk's change with cause `minted`, and
the queue's change (`bzh:publish-after-commit`). The pass logs a line per mint naming the cadence, the chore, the scope,
the chunk, and whether it was promoted, and a tally per tenant.

## The crash

A mint is one transaction, so the only window it opens is the one the routine-run mint already accepts: the work ref is
allocated before the transaction, and a crash between them burns one ref. The hub's crash-correctness record gains the
cadence mint beside the routine-run mint, recording the same single-transaction exemption, and the crash-point registry
gains no point. A crash after the commit loses only the board's push, and the board's next read shows the run. A hub
restarted mid-pass simply judges every errand again on its first pass back: one already minted has a live last run, and
is not due, and one whose moment fell while the hub was down is minted then ([due.md](./due.md) §A missed moment fires
late, once).

## The flag

Until the epic's last piece lands, the minting rides dark behind the hub flag `cadences`, declared in blizzard's flag
registry with the summary *cadences mint their errands' runs*. While it is off, the cadence pass runs, reads, and mints
nothing, and records, routes, verbs, and reads all answer as specified, since a flag hides behavior, never contract
(`bzh:flag-hides-behavior`). An operator may turn it on early with a `[flags]` override.
