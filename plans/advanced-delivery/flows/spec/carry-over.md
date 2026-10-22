# Carry-over

An installation that predates this epic keeps landing as it did, without a chunk being restarted by hand. The epic lands
as four chunks in the order the [spec index](./index.md#landing-order) gives, and every one of them leaves `master`
landing work and releasable. A release can be cut between any two of them, so an installation may take them one release
at a time or several in one, and it is carried over either way. The epic is delivered when the four have landed; the
slot's table, which they leave deprecated, is dropped by a cleanup after it
([below](#the-deprecation-then-the-cleanup)).

## The migrations

Two manual migrations (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, the first revised after the
forges epic's — the chain head when it merges — and the second after whatever heads the chain when the cleanup merges,
each frozen once it merges (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), proven on
Postgres as on SQLite (`tests/test_store_migrations.py`) with a colocated test of its own
(`bzh:colocated-migration-tests`), and with every constraint and index name inside Postgres's 63 characters
(`tests/test_migration_identifier_length.py`).

1. **The first, with the record.** It creates `flows`, `flow_submissions`, `flow_submission_flows`,
   `flow_submission_closures`, `hub_turns`, and `flow_facts`, with their indexes ([record.md](./record.md)). It writes
   no `landed` fact for a landing made before it ([record.md](./record.md#the-landed-event)). It only expands
   (`bzh:store-expand-contract`). Its `downgrade()` drops the six tables, and the line goes with them: a hub rolled back
   past it lands through the slot, which never left, and the flows' record of what happened while it stood is lost.
2. **The second, dropping the slot's table.** It drops `hub_exec_slot`, in the cleanup that follows the epic, never in
   one of its four chunks. By then nothing has acquired the slot for at least a release, so the table is empty on any
   hub that has booted since; a row still live belonged to a pass in a process that has stopped, which is what
   `release_orphaned_slots` assumed of it at every boot. Its rows go with it, and with them the `HUB_EXEC` intervals of
   chunks traced before the turns began ([record.md](./record.md#invariants)). Its `downgrade()` recreates
   `hub_exec_slot` empty, with its indexes as they stood: no build since the second chunk writes the table, and a hub
   rolled back further, onto the slot, needs nothing more, since no pass is in flight across a restart.

Neither migration admits a chunk, since repository resolution is the domain's and a migration does not run it.

## The deprecation, then the cleanup

Dropping a table is a contraction (`bzh:store-expand-contract`), which a release may make only once every supported
release has marked the table deprecated, so the slot's table leaves as anything released does: deprecated in one
release, dropped in a later one. The epic does the first half and is delivered without the second; none of its four
chunks is breaking.

**Deprecated in the second chunk**, the one that stops acquiring the slot: `hub_exec_slot`'s `Table` in
`hub/store/schema.py` carries `info={"deprecated": "<line>"}`, naming the line of the release that first ships the mark,
and `contracts/store/hub.json` with it; a `deprecated` note under `changes/unreleased/` names the table, with `surfaces`
`store` and a `removes-in` of the minor after that release. From then until the drop, no build writes the table; the
third chunk retires the code that did and keeps only the trace corpus's read of its past rows.

**Dropped by a follow-up cleanup** that belongs to no epic: a breaking change filed beside the epic and taken up only
once an operator decides a release may carry it, which is no sooner than a release carrying the deprecation has shipped,
and before the line its `removes-in` names is cut. It runs the second migration, removes `TracedHubExecSlot`, `_slots`,
and the corpus's read of the slot's rows, and raises `floor` in `wire/window.toml` to the line that deprecated the
table, as the window requires of the change that removes what a released minor deprecated (`docs/versioning.md`); from
it, a runner of an earlier line is refused registration. Its `removed` note `retires` the deprecation, is
`breaking: true` with `breaks` naming the table, and carries the `## Upgrade` section that says the table and the older
traces' `hub exec` intervals are gone; its commit is marked breaking, `feat!`. Those intervals, of every chunk traced
before the turns began, are an accepted loss: nothing carries them into `hub_turns`. The cleanup is taken up together
with the actions and forges epics' cleanups, which raise `floor` for the same reason, so that the fleet's oldest runners
are let go once, in one release, rather than three times. `blizzard-context`'s
`architecture/system-shape/store-facts.md` stops citing `hub_exec_slot.released_at` as its example of a released-at
column in the same change.

## At boot

`release_orphaned_slots` (`hub/app.py`, `HubNodeExecutor`) is replaced by `recover_turns`, run per open tenant before
the hub serves and before the flow driver starts. In the first chunk, while the slot still runs passes, boot runs
`release_orphaned_slots` and then `recover_turns` whole, whose first step finds no turn on a store no later chunk has
run; from the second, `recover_turns` runs alone and `release_orphaned_slots` is gone:

1. **Interrupted turns.** Every turn with no `ended_at` ended with the process that ran it, whether it crashed or shut
   down. It is ended `interrupted`, with a `turn-ended` fact on each flow it held and a `turn-interrupted` event
   (warning), and with the closure `withdrawal_for` answers when a lever deferred one to it ([levers.md](./levers.md)).
   A submission still current keeps its place and is eligible at once, and its next turn takes a fresh pass, which skips
   whatever its markers say is done, as a re-run does today.
2. **Chunks mid-delivery.** One reconcile ([verdict.md](./verdict.md#the-reconcile)) runs before the driver's first
   tick, after the actions epic's boot pass has escalated every chunk standing at a hub node with no action, so the
   reconcile finds those chunks `needs_human` and escalates none of them again. On the first boot of the epic's first
   chunk it submits every chunk `delivering` at a hub node, in the order each entered its node, so each heads its flows
   in the order it reached them, ahead of any chunk that arrives after the upgrade. A chunk that last polled `pending`
   keeps its poll history, so it is next when its poll is due, as it would have been under the slot. A chunk standing at
   a hub node while paused or `needs_human` at the upgrade is submitted by the reconcile of the tick after it is resumed
   or requeued. A chunk that landed some of its repositories before the upgrade submits holding only the flows of those
   it has left ([turns.md](./turns.md#what-a-step-holds)).

Both steps are idempotent, so a boot that dies midway is finished by the next. The second chunk's first boot finds the
line the first chunk kept, already in arrival order, and takes turns from it; a chunk that landed through the slot while
the first chunk stood has had its submission closed `superseded` by the reconcile, and its landing stands on its
markers.

Going back from the second chunk to the first touches no schema, since the second adds no revision: the operator
reinstalls the first chunk's build on the same store. A hub stopped mid-turn leaves its turns open, and the first
chunk's boot ends each of them `interrupted`, with the closure `withdrawal_for` answers for a chunk a lever moved, so
the line it reads is the one the second chunk left. The slot then runs every pass again, and a submission whose chunk
the slot lands is closed `superseded` by the next reconcile. A merge a cancelled pass had already made is found by the
slot's next pass, which reads the forge before it acts, as after a crash.

## What an operator meets

The second chunk is the one an operator notices, and it carries a `changed` note under `changes/unreleased/` and a
`docs/upgrade.md` section, linked by the note's `upgrade-guide`, for the release that first ships it. Landings on
different branches now run side by side, and a garden delivery or a review's findings no longer wait behind any landing.
A chunk's landing no longer needs its runner, and hub-advance no longer runs a step: it wakes the hub and answers
`ran: false`, saying in its detail where the visit stands. Stopping, restarting, or completing a chunk mid-pass lets
that pass finish and holds its branch until it does. A step cut off at its deadline three times running goes to a person
with the cause `hub-action-stuck`. And a chunk paused at a hub node no longer lands while it is paused: it waits for its
resume. That is the one change that asks something of the operator, and the section says it plainly: a chunk paused at a
hub node when the release is installed lands only once someone resumes it.

## Tests

The tests written against the slot follow it out across the second and third chunks. In the second, which stops
acquiring it, `tests/test_hub_command_node.py`, `tests/test_chunk_stop.py`, `tests/test_chunk_complete.py`,
`tests/test_attempt_coherence.py`, `tests/test_hub_store_seam.py`, and `tests/test_hub_wide_jobs_across_tenants.py`
assert turns and withdrawals where they asserted the slot's acquisition and release; `tests/trace_fixtures.py`,
`tests/test_trace_assembly.py`, `tests/test_trace_contract.py`, and `tests/test_hub_platform_tracing.py` build turns
where they built slot rows, keeping one case of a slot row from before the turns; and `tests/crash/test_kill9_sweep.py`
gains the `flow.` family and loses `hubnode.after-poll.before-slot-release`. In the third, which retires the slot's
code, `tests/test_hub_exec_slot_tenancy.py` and `tests/test_chunk_artifact_order_and_hub_exec_slot_store.py` lose their
slot cases, whose flow equivalents are the store tests of `flow_store`; `tests/test_pin_foundation.py` pins the flow
invariants in place of `OneLiveExecSlot`; and `tests/store_scan_allowlist.py` loses the slot's scans and
`tests/store_read_census.py` its slot recipes. The census gains a recipe per `IReadFlows` method in the first.

The end-to-end and crash tiers need one more test action than the actions epic ships. **`test.block`**, with required
argument `seconds`, blocks the `ActionLoop`'s thread in synchronous code for that long and then answers `success`; it
joins `test.answer` and `test.sequence` in the test-fenced catalogue, behind the same two fences, so no installation can
mint it. It is how the proofs below reach a pass that never comes back to an `await`.

## Proof

Each proof runs against the mock forge.

- **Side by side.** Two chunks landing into two repositories take turns at once; two into one branch take turns in
  arrival order; a garden delivery takes its turn while both land.
- **Pending yields.** A chunk whose CI is pending yields its branch to the next in line and is next again when its poll
  is due, and no sooner than the `next_poll_at` its pass answered with.
- **No deadlock, no starvation.** Two chunks each landing into the same two repositories, submitted in either order,
  both land, one after the other.
- **No runner needed.** A chunk whose runner is gone still lands, and moves on from `deliver` to `record-findings` and
  through it with no runner calling hub-advance.
- **Every path in.** A chunk reaching a hub node by a worker transition, a hub-to-hub transition, a restart, a
  migration, a resume, and a requeue after an escalation is each submitted within one tick, and none is submitted twice.
- **Nothing left holding a place.** A chunk escalated while in line is passed over while the line moves; requeued, its
  old submission closes `superseded` and the new visit joins at the back. A submission whose chunk has moved on without
  a withdrawal never takes a turn and is closed by the next reconcile.
- **Unroutable answers.** An answer the chunk's graph authors no edge for escalates the chunk at once, the turn ends
  `escalated`, and the branch moves on.
- **A turn that cannot end.** A turn whose ending write is made to fail is ended `interrupted` within two ticks, with
  its event, and its submission lands on its next turn. A tick made to raise after committing one turn and before
  returning leaves that turn with no task, and the sweep ends it within two ticks. A visit build that raises routes
  `failure` with its traceback, and one held past its bound ends `cut-off`. A `test.block` pass that blocks past the
  turn's deadline is cut off: its turn ends `cut-off` and `hub-action-overran` names the bound, an effect it hands over
  afterwards raises into the pass and writes nothing, a forge call it attempts after the deadline is refused before it
  is sent, an effect write in flight at the deadline commits before the turn's end, and the branch moves on. A visit cut
  off three times running escalates with `hub-action-stuck` on its fourth turn, and one interrupted by hub restarts does
  not.
- **Ground moved under a submission.** A repository whose base branch is edited while a chunk waits in its line lands
  nothing on either branch from the old submission: its next turn ends `superseded` without a pass, and the visit is
  resubmitted, at the next generation, at the back of the new branch's line.
- **Shutdown.** A hub stopped while a turn runs writes nothing for it; the next boot ends the turn `interrupted`, and
  the chunk lands on its next turn without merging twice.
- **Capabilities.** A repository whose forge's test adapter lacks a capability `land` requires escalates its chunk with
  `forge-capability-missing` from the driver's visit build, before any pass, and the turn ends `escalated`.
- **Levers.** A chunk stopped mid-turn keeps its branch busy until its pass ends and never lets a second landing onto it
  meanwhile, whatever the reconcile runs in between; its submission closes `stopped` at the turn's end, and closes
  `stopped` too when that end is made to fail and the sweep ends the turn instead. The merge it made in that pass has
  its marker, its `delivery_repo_landed` row, its `landed` fact, and its `flow-landed` event with its one member, and a
  second repository it had not reached stays unmerged. A stopped, restarted, or completed chunk waiting its turn leaves
  its flows. A chunk escalated in line, paused by its person, then requeued and resumed keeps its old place, passed
  over, until it is `delivering` again, then joins at the back.
- **Escalation is one write, one epoch up.** A hub killed between an escalation and its turn's end leaves neither
  without the other. Every cause that ends a turn — the bounce cap, a poll timeout, an unroutable answer,
  `hub-action-missing`, `forge-auth`, `forge-config`, `forge-capability-missing`, an action's own, and
  `hub-action-stuck` — opens the epoch above its visit's, and the chunk requeued from each joins the back of the line
  once and escalates again when it should.
- **Restart of the hub.** A hub killed at each `flow.` crash point ([turns.md](./turns.md#crash-points)), driven by the
  kill-9 sweep, recovers at boot and lands the chunk without merging twice.
- **One landing, one transaction.** A hub killed between a merge's effect and the driver's write of its marker, and
  after it, never leaves a `merged/<repo>` marker without its `delivery_repo_landed` row, its close intents, its
  `landed` fact, and its `flow-landed` event.
- **Carried over.** A fixture store captured before the first migration with chunks mid-delivery, one of them `pending`
  and one paused, is carried over through each of the four chunks in turn, and straight from before the first onto the
  fourth, as a release carrying several of them takes it, and lands every chunk with the outcomes and markers the slot
  produced, the paused one once it is resumed. The first migration's `downgrade()` runs against the store its upgrade
  produced and leaves a hub that boots and lands through the slot, and the second chunk's build walked back to the
  first's ends the turns it left open at boot and lands through the slot.
- **Every step a span.** A hub step taken by a turn has its `hub exec` interval, keyed by the turn id, in the second
  chunk and every one after it, and a step the slot ran before the turns keeps its own while the table stands.
- **Deprecated, then dropped.** From the second chunk `contracts/store/hub.json` marks `hub_exec_slot` deprecated and
  `blizzard:compat` reports nothing on any of the epic's four chunks; in the cleanup, the second migration's
  `downgrade()` leaves a hub that boots and lands through turns, and `blizzard:compat` admits the drop only with the
  deprecation released and `floor` raised.
- **Invariants.** `hub:one-turn-per-flow`, `hub:one-turn-per-visit`, and `hub:flow-line-standing` pass on the
  carried-over store and after every scenario above.
