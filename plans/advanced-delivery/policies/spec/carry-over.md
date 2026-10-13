# Carry-over

An installation that predates this slice lands exactly as it did, because every repository is given the policy it is
already landed by, and no graph changes. The slice ships in one release.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, chained after the flows slice's,
frozen once released (`bzh:frozen-revisions`), inside the portable surface (`bzh:sql-portable`), and proven on Postgres
as on SQLite with every name inside Postgres's 63 characters. Within every tenant it:

1. Adds `repositories.landing_policy`, filled `merge-on-checks` for every row, retired ones included, then tightened to
   `NOT NULL`; and `repositories.park_timeout_hours`, null.
2. Creates `repository_landing_order`, empty.
3. Adds `hub_node_poll.kind`, filled `pending` for every row, then tightened to `NOT NULL`.

No repository changes revision and no `config_changes` row is written: the policy each is given is the one it already
landed by. A chunk mid-delivery when the hub restarts resumes on `land` as a runner of policies, whose `merge-on-checks`
is the action it was running, with the markers it had already written.

## The graphs

No shipped graph changes. Their delivery nodes name `land`, and none authors `awaiting-merge`, which is now refused at
mint as `pending` is. The migration checks every stored graph version for a node authoring a choice by that name and
reports any it finds by graph, version, and tenant; none is expected, since no graph has had a reason to.

## Proof

Each proof runs against the mock forge, GitHub face and GitLab face alike.

- **Unchanged by default.** A fixture store captured before the migration carries over, every repository reads
  `merge-on-checks`, and every chunk lands with the outcomes and markers it would have before.
- **Fast-forward.** A chunk lands with no change opened, the base advanced to its proven revision. A base that moved is
  merged in mechanically and the new head proven before the advance; a conflicting base is `conflict`; a base moved
  between the compare and the advance refuses the advance, and the next pass lands it; a protected base is `failure`
  naming the protection.
- **Park.** A chunk opens its change, writes `awaiting-external-merge`, and waits past its node's `poll_timeout` without
  a bounce; a person's merge lands it at the merge revision within two minutes; a change closed unmerged is `failure`; a
  change unmerged past its timeout escalates `park-timeout`, and resolving the escalation resumes the wait.
- **Order.** A chunk reaching a library and its consumer lands the library first; with the library parked, the
  consumer's landing waits for the person's merge; with the consumer parked, its change opens only once the library has
  landed.
- **Capabilities.** Setting `fast-forward` on a GitLab repository is refused naming `refs.advance` and `branches.merge`;
  a policy whose capability vanished after it was set is refused at the next visit.
- **GitLab.** A chunk lands into a GitLab repository by `merge-on-checks` and by `park`, a rebased head passes the gate
  and a rebase that changed the work does not, and a chunk spanning a GitHub and a GitLab repository lands both.
- **Budget.** With the mock's `rate_limited` lever set, landings slow to `pending` and resume at the reset, parked polls
  and the sweeps are turned away first, and `forge-budget-low` and `forge-rate-limited` are each recorded once per
  window.
- **Invariants.** `hub:landing-order-acyclic` and the widened `hub:repository-references-in-tenant` pass on the
  carried-over store and after every scenario above.
