# Carry-over

An installation that predates this slice lands exactly as it did: every repository's `batch_size` is one, so every flow
lands its submissions alone, and no repository's policy changes. Batching and the forge's queue are an operator's
choice, repository by repository. The slice ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, chained after the policies
slice's, frozen once released (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), and proven on
Postgres as on SQLite with every name inside Postgres's 63 characters. It:

1. Adds `repositories.batch_size`, filled 1; `batch_wait_seconds`, filled 60; and `train_prefix`, filled
   `blizzard-train/`; each then tightened to `NOT NULL`.
2. Creates `flow_batches` and `flow_batch_members`, empty.
3. Adds `flow_facts.batch_id`, null for every row, and `hub_turns.batch_id`, null, then relaxes
   `hub_turns.submission_id` to nullable and adds the one-subject check.

No repository changes revision and no `config_changes` row is written. At first boot every repository is probed, so the
capabilities the hub offers it are known before any visit asks; a repository whose policy needs a probed capability it
turns out to lack — a `fast-forward` repository whose base only pull requests may change — is reported with a
`forge-capability-missing` event and refused at its next visit, as the forges slice refuses any missing capability.

## The mock

`blizzard-mock`'s forge grows what the proofs need, over its git backend:

- `POST` and `DELETE git/refs`.
- Check runs on any revision steerable by its content: a lever that fails a named check on every revision whose tree
  holds a given path, so a batch carrying one bad chunk fails and the halves without it pass. Today every revision but a
  pull request's head reads green, which can prove neither a failing batch nor a split.
- A branch-rules read (`rules/branches/{base}`) answering what the protection and merge-queue levers set.
- A merge queue: `merge-async` with `merge_action: merge_queue`, the GraphQL `mergeQueueEntry`,
  `RemovedFromMergeQueueEvent`, and `dequeuePullRequest`, ordered first come first served, proving each change with the
  steerable checks, with levers to remove a change for a conflict, a failure, or by hand.
- On its GitLab face, merge trains: the `merge_trains` routes, the project's `merge_trains_enabled`, and system notes
  for a removal.

## Proof

Each proof runs against the mock forge.

- **Unchanged by default.** A fixture store captured before the migration carries over and lands every chunk alone, with
  the outcomes and markers it would have before.
- **A batch.** Four chunks ready on one branch land in one advance; the base's revision is the batch's revision; each
  chunk's `merged/<repo>` names it; each change reads merged; one `flow-landed` carries all four.
- **A lone chunk, promptly.** One chunk ahead of its base and green lands without a train or a second run of checks.
- **Mechanical first.** A chunk behind its base joins a batch with no update to its own branch and no agent woken; one
  that conflicts only with a chunk ahead of it in the batch lands in the next batch or is ejected `conflict` there; one
  that conflicts with the base is ejected `conflict` at once.
- **Splitting.** A batch of eight with one bad chunk lands the seven and ejects the one `failure`, with findings, in no
  more proofs than halving needs; the seven land in arrival order.
- **Inherited.** A batch whose only failures the base shares ejects its oldest member `inherited-failure` and keeps the
  rest in line.
- **The combination is what lands.** A batch proven and then outpaced by a direct push to the base is rebuilt and proven
  again before the base advances; the base never holds a tree that was not proven.
- **Abandoned trains.** A repository whose CI ignores its train branches abandons the batch `unproven` and lands each
  member alone; a protected base abandons it `advance-refused`, retakes the probe, and lands each member alone.
- **Restart of the hub.** A hub killed after the advance and before the markers writes them at boot, merging nothing
  twice.
- **The forge's queue.** A `forge-queue` repository lands through the mock's merge queue on its GitHub face and its
  merge train on its GitLab face; each removal reason reaches the chunk as its verdict; a cleared queue re-enqueues; a
  stopped chunk's change is dequeued; with the queue switched off, the repository falls back and records
  `forge-capability-fallback`.
- **Invariants.** `hub:one-batch-per-flow`, `hub:batch-members-open`, `hub:turn-has-one-subject`, and the flows slice's
  invariants pass on the carried-over store and after every scenario above.
