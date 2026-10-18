# Carry-over

An installation that predates this epic keeps landing as it did, without a chunk being restarted by hand. The epic lands
as four chunks in the order the [spec index](./index.md#landing-order) gives, and every one of them leaves `master`
landing work; an installation that takes them one at a time, as the hosted hub does, is carried over at each.

## The migrations

Two manual migrations (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, the first revised after the
forges epic's — the chain head when it merges — and the second after the first, each frozen once it merges
(`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), proven on Postgres as on SQLite
(`tests/test_store_migrations.py`) with a colocated test of its own (`bzh:colocated-migration-tests`), and with every
constraint and index name inside Postgres's 63 characters (`tests/test_migration_identifier_length.py`).

1. **The first, with the record.** It creates `flows`, `flow_submissions`, `flow_submission_flows`,
   `flow_submission_closures`, `hub_turns`, and `flow_facts`, with their indexes ([record.md](./record.md)). It writes
   no `landed` fact for a landing made before it ([record.md](./record.md#the-landed-event)). Its `downgrade()` drops
   the six tables, and the line goes with them: a hub rolled back past it lands through the slot, which never left, and
   the flows' record of what happened while it stood is lost.
2. **The second, retiring the slot.** It drops `hub_exec_slot`. By then nothing has acquired the slot since the chunk
   before it, so the table is empty on any hub that has booted since; a row still live belonged to a pass in a process
   that has stopped, which is what `release_orphaned_slots` assumed of it at every boot. Its rows go with it, and with
   them the `HUB_EXEC` intervals of chunks traced before the upgrade ([record.md](./record.md#invariants)). Its
   `downgrade()` recreates `hub_exec_slot` empty, with its indexes as they stood: the chunk before it lands through
   turns and never touches the table, and a hub rolled back further, onto the slot, needs nothing more: no pass is in
   flight across a restart, so there is nothing to put in it.

Neither migration admits a chunk, since repository resolution is the domain's and a migration does not run it.

## At boot

`release_orphaned_slots` (`hub/app.py`, `HubNodeExecutor`) is replaced by `recover_turns`, run per open tenant before
the hub serves and before the flow driver starts. In the first chunk, while the slot still runs passes, boot runs
`release_orphaned_slots` and then `recover_turns`'s reconcile alone, since no turn exists to recover; from the second,
`recover_turns` runs whole and `release_orphaned_slots` is gone:

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

## Tests

The tests written against the slot follow it out, in the chunk that drops it. `tests/test_hub_exec_slot_tenancy.py` and
`tests/test_chunk_artifact_order_and_hub_exec_slot_store.py` lose their slot cases, and their flow equivalents are the
store tests of `flow_store`; `tests/test_hub_command_node.py`, `tests/test_chunk_stop.py`,
`tests/test_chunk_complete.py`, `tests/test_attempt_coherence.py`, `tests/test_hub_store_seam.py`,
`tests/test_hub_wide_jobs_across_tenants.py`, and `tests/test_pin_foundation.py` assert turns and withdrawals where they
asserted the slot's acquisition and release; `tests/trace_fixtures.py`, `tests/test_trace_assembly.py`,
`tests/test_trace_contract.py`, and `tests/test_hub_platform_tracing.py` build turns where they built slot rows; and
`tests/crash/test_kill9_sweep.py` gains the `flow.` family and loses `hubnode.after-poll.before-slot-release`.
`tests/store_scan_allowlist.py` loses the slot's scans and `tests/store_read_census.py` its slot recipes, gaining one
per `IReadFlows` method.

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
  turn's deadline is cut off: its turn ends `cut-off` with the bound named, an effect it hands over afterwards is
  refused, a forge call it attempts after the deadline is refused before it is sent, an effect write in flight at the
  deadline commits before the turn's end, and the branch moves on. A visit cut off three times running escalates with
  `hub-action-stuck` on its fourth turn, and one interrupted by hub restarts does not.
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
  and one paused, is carried over through each of the four chunks in turn and lands every chunk with the outcomes and
  markers the slot produced, the paused one once it is resumed. Each migration's `downgrade()` runs against the store
  its upgrade produced and leaves a hub that boots and lands on the code before it: through turns below the second, and
  through the slot below the first.
- **Invariants.** `hub:one-turn-per-flow`, `hub:one-turn-per-visit`, and `hub:flow-line-standing` pass on the
  carried-over store and after every scenario above.
