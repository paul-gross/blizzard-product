# Carry-over

An installation that predates this slice lands as it did, because every repository is given the policy it is already
landed by. The slice ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, chained after the flows slice's,
frozen once released (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), and proven on Postgres
as on SQLite with every name inside Postgres's 63 characters. Within every tenant it:

1. Adds `repositories.landing_policy`, filled `merge-on-checks` for every row, retired ones included, then tightened to
   `NOT NULL`; and `repositories.park_timeout_hours`, null.
2. Creates `project_repository_positions`, empty, so every project's repositories start unranked and land in slug order.
3. Adds `hub_node_poll.kind`, filled `pending` for every row, then tightened to `NOT NULL`.

No repository changes revision and no `config_changes` row is written: the policy each is given is the one it already
landed by. A chunk mid-delivery when the hub restarts resumes on `land` as a runner of policies, whose `merge-on-checks`
is the action it was running, with the markers it had already written.

Two things a carried-over fleet sees differently. A chunk landing in several repositories lands them one at a time in
slug order, where today's land script merges them in the order its commits name them, until an operator ranks the
project's repositories. And a person closing a fleet pull request unmerged now brings the chunk to a person as
`change-closed` only under `park`; under `merge-on-checks` a closed pull request is replaced by a fresh one, as today.

## The graphs

The shipped graphs' delivery nodes keep naming `land`, and their `poll_interval` goes from 30 to 60 seconds
([budget.md](./budget.md#the-cost-of-a-pass)), so each is minted anew on upgrade as any edited shipped graph is; chunks
pinned to an earlier version keep its interval. None authors `awaiting-merge` or `pending`, which are now refused at
mint. The migration checks every stored graph version for a node authoring a choice by either name and reports any it
finds by graph, version, and tenant; none is expected, since no graph has had a reason to.

## Proof

Each proof runs against the mock forge, GitHub face and GitLab face alike, unless it says otherwise.

- **Unchanged by default.** A fixture store captured before the migration carries over, every repository reads
  `merge-on-checks`, every project's repositories read unranked, and every single-repository chunk lands with the
  outcomes and markers it would have before.
- **A cheaper pass.** A merge-on-checks chunk whose checks run for ten polls spends, after its first pass, no more than
  one counted call a pass against the mock's rate headers, its other reads answered 304; its already-merged lookup is
  one filtered read.
- **Fast-forward.** A chunk ahead of its base lands with no change opened, the base advanced to its proven revision. A
  base that moved is merged in base first on a `-landing` branch, the merge's first parent the old tip, proven, and the
  base advanced to it; the landing branch is gone afterwards. A conflicting base is `conflict`; a base moved between the
  compare and the advance refuses the advance, and the next pass lands it; a protected base escalates `base-protected`.
- **Park.** A chunk opens its change, writes `awaiting-external-merge`, and waits past its node's `poll_timeout` without
  a bounce; a person's merge lands it at the merge revision within two minutes, found in the recently-closed listing
  however many other changes closed since; a change closed unmerged escalates `change-closed`, and requeuing it opens a
  fresh change; a change unmerged past its timeout escalates `park-timeout`, and requeuing it resumes the wait with the
  clock restarted.
- **Order.** A project ranking a library before its consumer lands the library first; with the library parked, the
  consumer's landing waits for the person's merge; with the consumer parked, its change opens only once the library has
  landed. A second project linking the same two repositories and ranking them the other way lands its own chunk consumer
  first. A project with one repository ranked and two unranked lands the ranked one first, then the others by slug.
  Unlinking a ranked repository and linking it again restores its position. The configuration document exports a
  project's `landing_order` and applies it back unchanged.
- **Capabilities.** Setting `fast-forward` on a GitLab repository is refused naming `refs.advance`; a policy whose
  capability vanished after it was set escalates `forge-capability-missing` at the next visit.
- **GitLab.** A chunk lands into a GitLab repository by `merge-on-checks` and by `park`; a rebase the hub requested
  passes the gate and one that changed the file set, or that nobody requested, does not; a change passing through
  `locked` lands rather than failing; a retried job that passed reads green; a merged-results pipeline is read and
  waited on; a chunk spanning a GitHub and a GitLab repository lands both. The same scenarios run in the GitLab contract
  suite against a real GitLab CE container ([gitlab.md](./gitlab.md#the-mock)).
- **Budget.** With the mock's rate lever lowering the remaining, parked polls and the sweeps are turned away first and
  landings last; with the remaining spent, or a secondary limit answered, landings slow to `pending` with their next
  poll at the reset and resume after it; `advance_ref`'s GraphQL call does not move the REST budget; `forge-budget-low`
  and `forge-rate-limited` are each recorded once per window.
- **Crash points.** Each of the policies' four crash points ([landing.md](./landing.md#crash-points)) is reached by a
  `blizzard:crash-sweep` scenario, and the chunk lands exactly once with its markers whole.
- **Invariants.** `hub:landing-position-on-link` and the widened `hub:repository-references-in-tenant` pass on the
  carried-over store and after every scenario above.
