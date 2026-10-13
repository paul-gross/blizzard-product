# Carry-over

An installation that predates this slice keeps landing exactly as it did, without a chunk being restarted by hand. The
slice ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, revised after the forges slice's,
frozen once released (`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`), proven on Postgres
as on SQLite (`tests/test_store_migrations.py`) with every constraint and index name inside Postgres's 63 characters
(`tests/test_postgres_identifier_length.py`). It:

1. Creates `flows`, `flow_submissions`, `flow_submission_flows`, `hub_turns`, and `flow_facts`
   ([record.md](./record.md)).
2. Drops `hub_exec_slot`. A row still live at upgrade belonged to a pass in a process that has stopped, which is what
   `release_orphaned_slots` assumed of it at every boot.

The migration admits no chunk, since repository resolution is the domain's and a migration does not run it.

## At boot

`release_orphaned_slots` (`hub/app.py`, `HubNodeExecutor`) is replaced by `recover_turns`, run per open tenant before
the hub serves and before the flow driver starts:

1. **Interrupted turns.** Every turn with no `ended_at` ended with the process that ran it. It is ended `interrupted`,
   with a `turn-ended` fact on each flow it held and a `turn-interrupted` event (warning). Its submission keeps its
   place and is eligible at once, and its next turn takes a fresh pass, which skips whatever its markers say is done, as
   a re-run does today.
2. **Chunks mid-delivery.** Every chunk that is `delivering` at a hub command node and whose visit has no submission —
   on the first boot of this release, every one of them — is submitted, in the order it entered that node. Since this
   runs before any new submission can arrive, each heads its flows in the order it reached them, ahead of any chunk that
   arrives after the upgrade. A chunk that last polled `pending` keeps its poll history, so it is next when its poll is
   due, as it would have been under the slot.

Both steps are idempotent, so a boot that dies midway is finished by the next.

## Proof

Each proof runs against the mock forge through the forges slice's seam.

- **Side by side.** Two chunks landing into two repositories take turns at once; two into one branch take turns in
  arrival order; a garden delivery takes its turn while both land.
- **Pending yields.** A chunk whose CI is pending yields its branch to the next in line and is next again when its poll
  is due.
- **No deadlock, no starvation.** Two chunks each landing into the same two repositories, submitted in either order,
  both land, one after the other.
- **No runner needed.** A chunk whose runner is gone still lands, and a stalled head never blocks a line.
- **Levers.** A chunk stopped mid-turn keeps its branch busy until its pass ends and never lets a second landing onto it
  meanwhile; a stopped, restarted, or completed chunk waiting its turn leaves its flows.
- **Restart of the hub.** A hub killed mid-turn ends the turn `interrupted` at boot and lands the chunk on its next turn
  without merging twice.
- **Carried over.** A fixture store captured before the migration with chunks mid-delivery, one of them `pending`, is
  carried over and lands every chunk with the outcomes and markers the slot produced.
- **Invariants.** `hub:one-turn-per-flow`, `hub:one-turn-per-visit`, and `hub:flow-line-standing` pass on the
  carried-over store and after every scenario above.
