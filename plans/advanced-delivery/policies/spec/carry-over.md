# Carry-over

An installation that predates this epic lands as it did, because every repository is given the policy it is already
landed by. The epic ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, chained after the flows epic's,
frozen once released (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), and proven on Postgres
as on SQLite with every name inside Postgres's 63 characters. Within every tenant it:

1. Adds `repositories.landing_policy`, filled `merge-on-checks` for every row, retired ones included, then tightened to
   `NOT NULL`; and `repositories.park_timeout_hours`, null.
2. Creates `project_repository_positions`, empty, so every project's repositories start unranked and land in slug order.
3. Creates `flow_submission_terms`, empty ([record.md](./record.md#when-a-visit-takes-its-terms)).
4. Adds `hub_node_poll.kind`, filled `pending` for every row, then tightened to `NOT NULL`.
5. Adds `close_intent_attempts.resume_at`, null, admitting the new `deferred` outcome
   ([budget.md](./budget.md#who-goes-first)).

No repository changes revision and no `config_changes` row is written: the policy each is given is the one it already
landed by. A chunk mid-delivery when the hub restarts has a submission with no terms; its next turn takes them, as a
first turn does, and they are `merge-on-checks` for every repository, so it resumes on `land` as a runner of policies,
whose `merge-on-checks` is the action it was running, with the markers it had already written.

Two things a carried-over fleet sees differently. A chunk landing in several repositories lands them one at a time in
slug order, where the `land` action merges them in the order its commits name them, until an operator ranks the
project's repositories. And a person closing a fleet pull request unmerged now brings the chunk to a person as
`change-closed` only under `park`; under `merge-on-checks` a closed pull request is replaced by a fresh one, as today.

## The graphs

The shipped graphs' delivery nodes keep naming `land`, and their `poll_interval` goes from 30 to 60 seconds
([budget.md](./budget.md#the-cost-of-a-pass)), so each is minted anew on upgrade as any edited shipped graph is; chunks
pinned to an earlier version keep its interval. None authors `awaiting-merge`, which joins `pending` in the actions
epic's `RESERVED_WAITS` and is from now refused at mint by that epic's rule. The migration checks every stored graph
version for a node authoring a choice named `awaiting-merge` and reports any it finds by graph, version, and tenant;
none is expected, since no graph has had a reason to.

## Proof

Each proof runs against the mock forge's GitHub face unless it says otherwise.

- **Unchanged by default.** A fixture store captured before the migration carries over, every repository reads
  `merge-on-checks`, every project's repositories read unranked, and every single-repository chunk lands with the
  outcomes and markers it would have before, including one mid-delivery at the upgrade.
- **A cheaper pass.** A merge-on-checks chunk whose checks run for ten polls spends, after its first pass, no more than
  one counted call a pass against the mock's rate headers, its other reads answered 304; its already-merged lookup is
  one filtered read.
- **Checks are read fresh.** A revision whose one check has completed green gains a second, pending check before the
  next pass, and the change is not merged until that one is green too.
- **No checks, no landing.** A change, or a fast-forward candidate, whose revision carries no check at all stays
  `pending` and bounces at its node's poll timeout.
- **Fast-forward.** A chunk ahead of its base lands with no change opened, the base advanced to its proven revision. A
  base that moved is merged in base first on a `-landing` branch, the merge's first parent the old tip, proven by its
  own checks on that branch, and the base advanced to it; the landing branch is gone afterwards. A conflicting base is
  `conflict`, its landing branch deleted; a base moved while the candidate proved finds the landing branch on a stale
  tip, deletes it, builds it again on the new tip, and lands that; a base moved between the compare and the advance
  refuses the advance, and the next pass lands it; a protected base escalates `base-protected`.
- **Park.** A chunk opens its change, writes `awaiting-external-merge`, and waits past its node's `poll_timeout` without
  a bounce; a person's merge — by merge commit, squash, or rebase — lands it at the merge revision within two minutes,
  found in the recently-closed listing however many other changes closed since, and read alone when it has fallen off a
  full listing; a change closed unmerged escalates `change-closed`, and requeuing it opens a fresh change; a change
  merged while its chunk sat escalated lands the requeued visit at once; a change unmerged past its timeout escalates
  `park-timeout`, and requeuing it resumes the wait with the clock restarted.
- **Order.** A project ranking a library before its consumer lands the library first; with the library parked, its
  change opens only once the consumer is proven, and the consumer lands after the person's merge; with the consumer
  parked, its change opens only once the library has landed. With a parked library ranked first and the consumer's
  checks still running, the library's change stays unopened and the pass answers `pending`; once the change is open, a
  consumer whose base moved keeps the pass at `awaiting-merge`. A second project linking the same two repositories and
  ranking them the other way lands its own chunk consumer first. A project with one repository ranked and two unranked
  lands the ranked one first, then the others by slug. Unlinking a ranked repository and linking it again restores its
  position. The configuration document exports a project's `landing_order` and applies it back unchanged.
- **Terms.** A repository's policy changed from `merge-on-checks` to `park` while a chunk waits in line behind another
  lands that chunk parked; changed after the chunk's first turn, the chunk lands by the policy it started with, and its
  next visit after a requeue takes the new one. A project reordered while a chunk waits behind another lands it in the
  new order. A turn interrupted after it took its terms keeps them on its retry.
- **Capabilities.** Setting `fast-forward` on a repository whose forge kind's capabilities, in a table the test hands
  `ConfigAuthoring`, lack `refs.advance` is refused naming it; a policy whose capability vanished after it was set
  escalates `forge-capability-missing` at the next visit.
- **Across repositories.** A chunk spanning a `fast-forward` repository and a `merge-on-checks` repository on a second
  GitHub server lands both in its project's order, each through its own forge and credential; with the second refusing
  after the first has landed, the first stands and the chunk reconciles the second.
- **Budget.** With the mock's rate lever lowering the remaining, parked polls and the sweeps are turned away first and
  landings last; with the remaining spent, or a secondary limit answered, landings slow to `pending` with their next
  poll at the reset and resume after it; a close intent turned away is deferred to the reset with its backoff untouched;
  `forge-budget-low` and `forge-rate-limited` are each recorded once per window.
- **Crash points.** Each of the policies' six crash points ([landing.md](./landing.md#crash-points)) is reached by a
  `blizzard:crash-sweep` scenario, and the chunk lands exactly once with its markers whole.
- **Invariants.** `hub:landing-position-on-link`, `hub:submission-terms-standing`, and the widened
  `hub:repository-references-in-tenant` pass on the carried-over store and after every scenario above.
