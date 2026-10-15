# Carry-over

An installation that predates this epic lands exactly as it did: every repository's `batch_size` is one, so every flow
lands its submissions alone, and no repository's policy changes. Batching and the forge's queue are an operator's
choice, repository by repository. The epic ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, chained after the policies
epic's, frozen once released (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), and proven on
Postgres as on SQLite with every name inside Postgres's 63 characters. It:

1. Adds `repositories.batch_size`, filled 1; `batch_wait_seconds`, filled 60; `train_prefix`, filled `blizzard-train/`;
   and `staging_prefix`, filled `blizzard-staging/`; each then tightened to `NOT NULL`.
2. Creates `flow_batches` and `flow_batch_members`, empty, with their indexes.
3. Adds `flow_facts.batch_id`, null for every row, relaxes `flow_facts.submission_id` and `chunk_id` to nullable, and
   adds the fact-subject check.
4. Adds `hub_turns.batch_id`, null, then relaxes `hub_turns.submission_id` to nullable and adds the one-subject check.

No repository changes revision and no `config_changes` row is written. At first boot every repository is probed, so the
capabilities the hub offers it are known before any visit asks; a probe the forge cannot answer at boot is retried at
the next visit's check, which answers `pending` meanwhile. A repository whose policy needs a probed capability it turns
out to lack — a `fast-forward` repository whose base only pull requests may change — is reported with a
`forge-capability-missing` event and escalated to a person at its next visit, as the forges epic escalates any missing
capability.

## The mock

`blizzard-mock`'s forge grows what the proofs need, over its git backend:

- `POST` and `DELETE git/refs`, and `protected` on a branch read.
- Check runs on any revision steerable by its content: a lever that fails a named check on every revision whose tree
  holds a given path, so a batch carrying one bad chunk fails and the halves without it pass. Today every revision but a
  pull request's head reads green, which can prove neither a failing batch nor a split.
- Levers on those checks: one that runs them only on branches under a given prefix, so staging branches run none; one
  that leaves a check queued forever; one that turns the base's own checks red until cleared; and one that fails a named
  check once and passes it on its re-run.
- A branch-rules read (`rules/branches/{base}`) answering what the protection, linear-history, and merge-queue levers
  set, and GraphQL `repository.mergeQueue(branch:)` answering the merge-queue lever.
- A merge queue: GraphQL `enqueuePullRequest`, `mergeQueueEntry`, `RemovedFromMergeQueueEvent`, and
  `dequeuePullRequest`, ordered first come first served, proving each change with the steerable checks, with levers to
  remove a change for a conflict, a failure, a timeout, or by hand.
- On the GitLab face the `gitlab` epic built, merge trains: the `merge_trains` routes, the project's
  `merge_trains_enabled`, and system notes for a removal.
- A lever that answers a forge read with a 5xx, so a probe can fail.

## Proof

Each proof runs against the mock forge.

- **Unchanged by default.** A fixture store captured before the migration carries over and lands every chunk alone, with
  the outcomes and markers it would have before.
- **A batch.** Four chunks ready on one branch land in one advance; the base's revision is the batch's revision; each
  chunk's `merged/<repo>` names it; each change reads merged; one `flow-landed` carries all four in `members`.
- **One run of checks per batch.** The train branch is pushed once, at the batch's finished tree; no check runs on the
  staging branch.
- **A lone chunk, promptly.** One chunk that holds its base and is green on its own revision lands by an advance to that
  revision, without a train or a second run of checks.
- **Mechanical first.** A chunk behind its base joins a batch with no update to its own branch and no agent woken; one
  that conflicts only with a chunk ahead of it in the batch lands in the next batch or is ejected `conflict` there; one
  that conflicts with the base is ejected `conflict` at once. A chunk whose own checks have already failed is answered
  `failure` and never joins.
- **Splitting.** A batch of eight with one bad chunk lands the seven and ejects the one `failure`, with findings, in no
  more than seven proofs; the seven land in arrival order. A check that fails once and passes on its re-run ejects no
  one.
- **Inherited, and held.** A batch whose only failures the base shares ejects its oldest member `inherited-failure`,
  holds the flow, forms no further batch while the base is red, and resumes once the base's checks pass; no second
  member is ejected for the same red base.
- **Waiting is not a timeout.** A split that runs past the node's `poll_timeout` turns no member back, since its members
  answer `awaiting-queue`.
- **A proof that never finishes.** A check left queued forever is re-run once at `TRAIN_PROOF_TIMEOUT`, then the batch
  is abandoned `proof-timed-out` and formed afresh.
- **Order and readiness.** A chunk reaching two batching repositories in one project lands them one after the other, in
  that project's landing order; a chunk reaching a batching and a non-batching repository lands each in order; two
  projects ordering the same pair oppositely each land by their own order.
- **Ejected in one train while in another.** A chunk ejected from one repository's batch while it sits in another's
  batch, still proving, abandons that batch, which is rebuilt and lands without it.
- **Stopped mid-proof.** A member stopped while its batch proves abandons the batch, which lands without it; a member
  stopped after the advance is recorded landed.
- **The combination is what lands.** A batch proven and then outpaced by a direct push to the base is rebuilt and proven
  again before the base advances; the base never holds a tree that was not proven.
- **Abandoned trains.** A repository whose CI ignores its train branches abandons the batch `unproven` and lands each
  member alone; a protected base abandons it `advance-refused`, remembers the refusal, and lands each member alone, and
  no later scheduled probe offers `refs.advance` again until one is taken on demand.
- **Probes.** A probe answered by a 5xx leaves the last answer standing; a repository never probed answers `pending`
  until a probe succeeds; a base requiring linear history cannot batch.
- **Restart of the hub.** Each of the train's crash points, reached by the kill-9 sweep, recovers on the next pass: a
  hub killed after the advance and before the record writes the record, merging nothing twice; one killed after an
  enqueue writes `queued/<repo>/<n>` without enqueuing again.
- **The forge's queue.** A `forge-queue` repository lands through the mock's merge queue on its GitHub face and its
  merge train on its GitLab face; a conflict and failed checks reach the chunk as its verdict; a person's removal and a
  timed-out queue escalate to a person; a cleared queue re-enqueues; a stopped chunk's change is dequeued; with the
  queue switched off, the repository lands by `merge-on-checks`, unbatched, and records `forge-capability-fallback`.
- **Invariants.** `hub:one-batch-per-flow`, `hub:batch-members-open`, `hub:turn-has-one-subject`,
  `hub:flow-fact-has-subject`, and the flows epic's invariants pass on the carried-over store and after every scenario
  above.
