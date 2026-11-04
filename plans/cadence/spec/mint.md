# Minting an errand

## The sweep

The hub looks over its cadences in a seventh sweep, `SweepName.CADENCE` (`hub/domain/sweeps/names.py:12`), every 60
seconds: an arm in `_sweep_interval` (`hub/app.py:254`) and an entry in the `SweepRoster` built in
`hub/composition.py:1762`. `[sweeps.cadence] mode` is declared with the rest by the loop in `hub/config_keys.py:112`, so
`contracts/config/hub.json` is regenerated. The pass is `CadenceSweep` in `hub/domain/cadence/`, patterned on
`CloseDrainSweep` (`hub/domain/work_items/closure.py:306`): it reads the corpus of every open tenant once through
`HubScopedReads`, then does each tenant's work through that tenant's own services (`bzh:sweep-acts-in-row-tenant`), each
tenant inside its own catch (`bzh:pass-isolates-tenants`), with the pass added to the poisoned-tenant fixture
`tests/pass_isolation.py`. It serves one tenant on request as well, so `POST /api/sweeps/cadence/passes` and its admin
twin step it. It is a pass that acts on work as it comes due, which `bzh:probe-gated-pass` leaves without a probe or a
floor: time passing is its only signal.

A pass, for each tenant:

1. Reads one snapshot (`bzh:pass-reads-one-snapshot`): the tenant's cadences not retired, with their chores and listed
   scopes; the routines they name, by name, in bulk; the enabled graphs among those routines' graph names
   (`IReadGraphRepository.enabled_names`); the scopes they name and each routine's linked scopes; and each errand's last
   run with its chunk's status facts ([due.md](./due.md) §The last run).
2. Works out every errand and judges it ([due.md](./due.md) §The due rule), skipping any errand the cadence cannot run
   ([surfaces.md](./surfaces.md) §Needs attention).
3. Mints each due errand in a transaction of its own ([§The locked write](#the-locked-write)), so one errand's refusal
   or conflict never holds back another.

A pass is bounded: it mints at most `CADENCE_PASS_LIMIT` errands, 50, shared fairly across tenants by the `fair_share`
the close drain uses, starting from a different tenant each pass. An errand left over is still due on the next pass a
minute later, and is minted once there. A fleet switched off for a week comes back to a few passes of minting, never to
a pile of runs.

## The mint

An errand is minted through `RunService.run` (`hub/domain/garden/runs/run.py:193`), the path a hand-started run takes,
and nothing goes around it: the same refusals in the same order, the same settling of a `delta` with no baseline down to
`full`, recorded as downgraded, the same charge with the chore's note as its *This run* section, the same title, and the
work minted in the routine's project, which is the cadence's. `RunService.run` gains two optional arguments: the
errand's mark, and whether to promote. A hand-started run passes neither and is minted exactly as it is today.

The delta baseline is still the routine's own over that scope, read from `finding_sets`
(`hub/store/internal/finding_store.py:590`), which keys by routine and scope rather than by run. A monthly full run from
one chore therefore sets the baseline the next weekly delta from another chore reads, as the plan intends.

## The errand's mark

`work_item_runs` (`hub/store/schema.py:2953`) gains two nullable columns, `cadence_id` and `chore_id`, and an index on
`(tenant_id, chore_id, scope_id)` for the newest-per-errand read. `create_run_with_chunk` writes them in the transaction
that writes the run's identity row (`hub/store/internal/run_context_store.py:22`), so no run exists without its mark,
and no mark exists without its run. A hand-started run leaves both null. `RunContext`
(`hub/domain/garden/run_context.py`) and `RunIdentity` (`hub/domain/garden/runs/history.py:157`) carry the mark, so a
run's view can name the cadence and chore that minted it ([surfaces.md](./surfaces.md) §Runs). The columns arrive in an
additive hub revision of their own, landed with the sweep ([carry-over.md](./carry-over.md)).

## The author

`WorkItemAuthor` (`hub/domain/chunk/model.py:70`) is a person, `user`, or a fleet node, `fleet`. A cadence's run is
neither, so it gains a third kind, `cadence`, carrying the cadence's id and the chore's id. The author is stored as JSON
(`hub/store/schema.py:2921`), so the store needs no change. Every view that answers an author's kind declares it open
(`bzh:open-response-vocabulary`). Where one is closed today, the kind it gains is a new value, which a released client
must already tolerate, and that view is made open in the change that adds it.

## Promoting

A chore that promotes admits its run to the ready queue in the minting transaction itself: the `chunk_promoted` fact and
the run's tail position in the ready list, as `PromoteService.promote` records them
(`hub/domain/operations/promote.py:83`, `queue.record_promote_with_position`), are written beside the run. A run that is
minted promoted never rests `not_ready`, even for an instant a runner could see. A chore with `promote: false` mints its
run exactly as a hand-started run is minted, resting behind the gate until someone promotes it.

This is the one admission to the queue beside an operator's. `blizzard-context`'s `domain/work/statuses.md` §Promotion
and demotion and `domain/routines-and-scopes.md` §A run is an act of the pair say so: a run rests behind the promote
gate unless the cadence chore that minted it promotes. `domain/findings-and-proposals.md`'s accepted proposal still
rests behind the gate, and is unchanged.

## The locked write

Each mint is an exactly-one-wins decision (`bzh:write-reestablishes-read`). Two passes, the timed one and one requested
on the route, may judge the same errand due on the same snapshot. The mint therefore takes a keyed lock on the errand,
`lock_keys(conn, "cadence-errand", [f"{chore_id}:{scope_id}"])` (`hub/store/locking.py`). Under that lock, and in the
same transaction, it reads the errand's last run again, and mints nothing when that run is newer than the one the pass
judged or is unfinished. A lost race is not an error: the pass logs it at debug and moves on. Proven on Postgres by
`blizzard:postgres-uow-concurrency`, whose lock order the new key joins.

## Refused, conflicted, faulted

An errand the snapshot showed runnable may stop being so before its mint, because its routine was retired or its scope
unlinked a moment earlier. `RunService.run` then refuses it as it would refuse an operator. The pass logs the refusal,
mints nothing, and the next read shows the attention it needs. An `IngestConflict`, which the hand-started route answers
`409`, is handled the same way, for that errand alone. Any other exception from one errand is caught and logged, and the
pass carries on; a tenant whose snapshot cannot be read at all is that tenant's `TenantFault`, the other tenants
untouched. None of these is latched or retried with backoff: the next pass, a minute on, judges the errand afresh
(`bzh:lane-contract`).

## After the commit

Each mint, once committed, publishes what the hand-started route publishes (`hub/api/routines.py:646`): the chunk's
`ChunkChanged` with cause `minted`, and `queue_changed` (`bzh:publish-after-commit`). The pass logs a line per mint
naming the cadence, the chore, the scope, the chunk, and whether it was promoted, and a tally per tenant.

## The crash

A mint is one transaction, so the only window it opens is the one the routine-run mint already accepts: the work ref is
allocated before the transaction, and a crash between them burns one ref. `architecture/crash-correctness/hub.md` gains
§The cadence mint, recording the single-transaction exemption beside §The routine-run mint, and the crash-point registry
gains no point. A crash after the commit loses only the board's push, and the board's next read shows the run. A hub
restarted mid-pass simply judges every errand again on its next pass: one already minted has a live last run, and is not
due.

## The flag

Until the epic's last piece lands, the minting rides dark behind the hub flag `cadences`, declared in
`foundation/flags/registry.toml` with the summary *cadences mint their errands' runs*. While it is off, the cadence pass
runs, reads, and mints nothing, and records, routes, verbs, and reads all answer as specified, since a flag hides
behavior, never contract (`bzh:flag-hides-behavior`). An operator may turn it on early with a `[flags]` override, as
`docs/flags.md` describes.
