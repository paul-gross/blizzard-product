# Carry-over

An installation that predates this slice keeps landing as it did, without a chunk being restarted by hand. The slice
ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, revised after the actions
slice's, frozen once released (`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`), proven on
Postgres as on SQLite (`tests/test_store_migrations.py`) with every constraint and index name inside Postgres's 63
characters (`tests/test_postgres_identifier_length.py`). It:

1. Creates `flows`, `flow_submissions`, `flow_submission_flows`, `flow_submission_closures`, `hub_turns`, and
   `flow_facts`, with their indexes ([record.md](./record.md)).
2. Drops `hub_exec_slot`. A row still live at upgrade belonged to a pass in a process that has stopped, which is what
   `release_orphaned_slots` assumed of it at every boot. Its rows go with it, and with them the `HUB_EXEC` intervals of
   chunks traced before the upgrade ([record.md](./record.md#invariants)).

The migration admits no chunk, since repository resolution is the domain's and a migration does not run it.

## At boot

`release_orphaned_slots` (`hub/app.py`, `HubNodeExecutor`) is replaced by `recover_turns`, run per open tenant before
the hub serves and before the flow driver starts:

1. **Interrupted turns.** Every turn with no `ended_at` ended with the process that ran it. It is ended `interrupted`,
   with a `turn-ended` fact on each flow it held and a `turn-interrupted` event (warning). Its submission keeps its
   place and is eligible at once, and its next turn takes a fresh pass, which skips whatever its markers say is done, as
   a re-run does today.
2. **Chunks mid-delivery.** One reconcile ([verdict.md](./verdict.md#the-reconcile)) runs before the driver's first
   tick. On the first boot of this release it submits every chunk `delivering` at a hub node, in the order each entered
   its node, so each heads its flows in the order it reached them, ahead of any chunk that arrives after the upgrade. A
   chunk that last polled `pending` keeps its poll history, so it is next when its poll is due, as it would have been
   under the slot. A chunk standing at a hub node while paused or `needs_human` at the upgrade is submitted by the
   reconcile of the tick after it is resumed or requeued.

Both steps are idempotent, so a boot that dies midway is finished by the next.

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
- **A turn that cannot end.** A turn whose ending write is made to fail is ended `interrupted` within one tick, with its
  event, and its submission lands on its next turn.
- **Levers.** A chunk stopped mid-turn keeps its branch busy until its pass ends and never lets a second landing onto it
  meanwhile; a stopped, restarted, or completed chunk waiting its turn leaves its flows.
- **Restart of the hub.** A hub killed at each `flow.` crash point ([turns.md](./turns.md#crash-points)), driven by the
  kill-9 sweep, recovers at boot and lands the chunk without merging twice.
- **One landing, one transaction.** A hub killed between a merge's effect and the driver's write of its marker, and
  after it, never leaves a `merged/<repo>` marker without its `delivery_repo_landed` row, its `landed` fact, and its
  `flow-landed` event.
- **Carried over.** A fixture store captured before the migration with chunks mid-delivery, one of them `pending` and
  one paused, is carried over and lands every chunk with the outcomes and markers the slot produced, the paused one once
  it is resumed.
- **Invariants.** `hub:one-turn-per-flow`, `hub:one-turn-per-visit`, and `hub:flow-line-standing` pass on the
  carried-over store and after every scenario above.
