# Carry-over

An installation that predates this epic lands as it did, because every repository is given the policy it is already
landed by. The epic lands in the steps its [spec index](./index.md#landing-order) orders. The instance runs a pinned
build and moves only when an operator moves it, but a release can be cut from `master` between any two steps, so each
step must stand on its own as a release: landing as before, judged additive against every supported release, and able to
walk its store back to the step before it. Three of the steps carry a migration.

## The migrations

Three manual migrations (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, chained after the flows
epic's. None is edited once it merges, since any build of `master` — a dogfood build, or a release cut between two steps
— may already have run it, and each declares the tables it reshapes as frozen literals (`bzh:frozen-revisions`). Each
sits inside the portable surface (`bzh:sql-portable`), is proven on Postgres as on SQLite with every name inside
Postgres's 63 characters, and has a colocated test of its upgrade and its downgrade (`bzh:colocated-migration-tests`).

**The budget's**, with the budget's step, adds `close_intent_attempts.resume_at`, null, admitting the new `deferred`
outcome ([budget.md](./budget.md#who-goes-first)). Its downgrade deletes the `deferred` rows, which the older hub's
`close_intent_is_due` would misread as failures, and drops the column.

**The policies'**, with the step that makes `land` a runner of policies, within every tenant:

1. Adds `repositories.landing_policy` with a server default of `merge-on-checks`, fills it for every row, retired ones
   included, and tightens it to `NOT NULL`, keeping the default; and adds `repositories.park_timeout_hours`, null.
2. Creates `project_repository_positions`, empty, so every project's repositories start unranked and land in slug order.
3. Creates `flow_submission_terms`, empty ([record.md](./record.md#when-a-visit-takes-its-terms)).
4. Adds `hub_node_poll.kind` with a server default of `pending`, fills it for every row, and tightens it to `NOT NULL`,
   keeping the default; and adds `hub_node_poll.last_polled_at`, null ([landing.md](./landing.md#awaiting-a-merge)).

The two server defaults are what keep the revision an expansion: a column tightened to `NOT NULL` on a table a supported
release ships is a contraction without one (`bzh:store-expand-contract`), and `blizzard:compat` reports it as such. The
hub still writes both columns explicitly; the defaults speak only for a row written without them.

No repository changes revision and no `config_changes` row is written: the policy each is given is the one it already
landed by. A chunk mid-delivery when the hub restarts has a submission with no terms; its next turn takes them, as a
first turn does, and they are `merge-on-checks` for every repository, so it resumes on `land` as a runner of policies,
whose `merge-on-checks` is the action it was running, with the markers it had already written.

Its downgrade drops `hub_node_poll.kind` and `last_polled_at`, drops `flow_submission_terms` and
`project_repository_positions` — every project's landing order is lost, and chunks land in the order their commits name
them again — and drops the two repository columns. It needs no guard of its own: the only policy and the only wait a
store at this revision can hold are the ones the older hub lands by, and anything the next step adds is cleared by that
step's downgrade first.

**The two policies'**, with the step that adds `fast-forward` and `park`, changes no schema, since the columns the
policies' migration added already hold any policy and any wait. It exists so that a store a hub of this step has written
cannot be served by a hub of the step before it until its downgrade has run. Its upgrade checks every stored graph
version for a node authoring a choice named `awaiting-merge`, which this step reserves, and reports any it finds by
graph, version, and tenant ([The graphs](#the-graphs)). Its downgrade is the guard ([Walking back](#walking-back)).

## Walking back

A hub refuses to serve a store whose revision is not its own, naming the migrate command
(`foundation/store/migrations.py`, `MigrationRunner.check_current`), so a store moves back to an older release only
through the downgrades of every revision between them, each run on the newer build. A downgrade that would leave the
older hub facing what it cannot read, or would quietly change how a repository lands, refuses before it changes
anything, naming what the operator must settle, as `20261012_1000_hosts.py`'s `_refuse_hosted_runners` refuses while a
runner still stands under a host. That mechanism is why the two policies' step carries a revision even though it changes
no schema: without one, a release carrying only the step before it would meet `fast-forward`, `park`, and
`awaiting-merge` with nothing between them to stop it.

The two policies' downgrade refuses while any repository's `landing_policy` is not `merge-on-checks`, naming each one
and its tenant, and while any open submission's terms name another policy, naming its chunk. The older hub lands every
repository by merging once checks pass, so a repository set to `park` would have its pull requests merged without the
person it was waiting on; the operator sets each repository back by hand, then requeues or stops each chunk still
landing by the old terms, whose next visit takes `merge-on-checks` afresh, and the choice to stop honouring a policy is
theirs. Past that check it deletes the terms of closed submissions that name another policy, which nothing reads once a
submission has closed, and the `awaiting-merge` rows of `hub_node_poll`, which the older hub's poll timeout would count
as pending polls and bounce a chunk at once. An escalation cause this epic adds is stored as text, which the older hub
reads and shows as it stands.

Each other step walks back as its own section says: the board's and the seam's carry no store change, the budget's
downgrade deletes its deferred attempts, and the policies' drops what it added.

## Compatibility

Every step is additive against every supported release (`bzh:released-surface-additive`), as `blizzard:compat` judges
it, and none is breaking:

- **The seam and the cheaper pass** change no contract surface.
- **The budget** adds a nullable column and an `outcome` value, two event kinds — `EventLoggedPayload.kind` is a plain
  string in every supported release's document — and a response field on the forges epic's `GET /api/forges/{forge}`.
- **The runner of policies** adds two tables and four columns, the two tightened ones with their server defaults;
  optional request fields; response fields, `landing_policy` among them, an open response vocabulary
  (`bzh:open-response-vocabulary`); and two routes.
- **The two policies** add values to `landing_policy` on both sides, to the flows epic's place `state`, which that epic
  made open, and to `EscalationCause`, which no operation of a supported release's document reaches — its
  `ChunkEscalationView.cause` is a plain string — and response fields on `PendingView`, its wait's kind an open
  vocabulary.
- **The board** changes no contract surface.

A finding none of these expects is made additive before the step merges: the old form stays beside its replacement,
marked deprecated, as `bzh:store-expand-contract` and `bzh:released-surface-additive` ask. Its removal is no part of
this epic, which is delivered when its additive steps land: it waits for a cleanup change of its own, marked breaking,
outside the epic. None of the six steps removes anything, so no such change is owed.

Two things a carried-over fleet sees differently. A chunk landing in several repositories lands them one at a time in
slug order, where the `land` action merges them in the order its commits name them, until an operator ranks the
project's repositories. And a person closing a fleet pull request unmerged now brings the chunk to a person as
`change-closed` only under `park`; under `merge-on-checks` a closed pull request is replaced by a fresh one, as today.

## The graphs

The shipped graphs' delivery nodes keep naming `land`, and their `poll_interval` goes from 30 to 60 seconds
([budget.md](./budget.md#the-cost-of-a-pass)), so each is minted anew on upgrade as any edited shipped graph is; chunks
pinned to an earlier version keep its interval. None authors `awaiting-merge`, which joins `pending` in the actions
epic's `RESERVED_WAITS` with the two policies' step and is from then refused at mint by that epic's rule. That step's
migration checks every stored graph version for a node authoring a choice named `awaiting-merge` and reports any it
finds by graph, version, and tenant; none is expected, since no graph has had a reason to.

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
  tip, deletes it, builds it again on the new tip, and lands that; a landing branch holding an older revision of the
  chunk is rebuilt the same way; a base moved between the compare and the advance refuses the advance, and the next pass
  lands it; a crash after the advance, with the base moved on by another chunk before the next pass, records the
  candidate rather than the base's tip; a protected base, and a ruleset refusing the `-landing` branch, each escalate
  `branch-protected`.
- **Merge method.** A `merge-on-checks` repository whose forge admits no merge commits escalates `merge-method-refused`
  at its first merge rather than waiting out its poll timeout.
- **Park.** A chunk opens its change, writes `awaiting-external-merge`, and waits past its node's `poll_timeout` without
  a bounce, its waits coalesced into one `hub_node_poll` row; a person's merge — by merge commit, squash, or rebase —
  lands it at the merge revision by the first poll after it, within two minutes and one driver tick, found in the
  recently-closed listing however many other changes closed since, and read alone when it has fallen off a full listing;
  a change closed unmerged escalates `change-closed`, and requeuing it opens a fresh change; a change closed while its
  chunk sat escalated, then requeued, opens a fresh change without escalating again; a change merged while its chunk sat
  escalated lands the requeued visit at once; a change unmerged past its timeout escalates `park-timeout`, and requeuing
  it resumes the wait with the clock restarted and the board reading "Awaiting your merge" again. While the change is
  open, a later repository whose base moved makes no forge call, and is brought up to its base and proven again after
  the merge. A chunk parked in two repositories gives the second change its whole timeout, measured from when it opened.
- **Order.** A project ranking a library before its consumer lands the library first; with the library parked, its
  change opens only once the consumer is proven, and the consumer lands after the person's merge; with the consumer
  parked, its change opens only once the library has landed. With a parked library ranked first and the consumer's
  checks still running, the library's change stays unopened and the pass answers `pending`; once the change is open, a
  consumer whose base moved keeps the pass at `awaiting-merge`, and a consumer that showed a failure before the change
  opened does not decide the pass while it is open. A second project linking the same two repositories and ranking them
  the other way lands its own chunk consumer first. A project with one repository ranked and two unranked lands the
  ranked one first, then the others by slug. Unlinking a ranked repository and linking it again restores its position.
  The configuration document exports a project's `landing_order` and applies it back unchanged, writing no fact the
  second time. A project whose positions have run out of room between two neighbours is renumbered by the next move, and
  keeps its order.
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
  landings last, a landing chunk and a parked one on the same repository each at its own priority; with the remaining
  spent, or a secondary limit answered, landings slow to `pending` with their next poll at the reset and resume after
  it; a close intent turned away is deferred to the reset with its backoff untouched; `forge-budget-low` and
  `forge-rate-limited` are each recorded once per window.
- **Downgrades.** Each migration's downgrade runs on a store the upgrade produced. The two policies' downgrade refuses
  while a repository is not `merge-on-checks`, naming it, and while a parked chunk's open submission still holds `park`
  terms, naming the chunk; once both are settled it leaves a store a hub of the step before lands from, with no chunk
  bounced for an `awaiting-merge` row. A hub of the step before refuses that store until the downgrade has run.
- **Compatibility.** `blizzard:compat` passes at every step against every supported release, and `blizzard:rehearsal`
  walks each supported release's store up to the step and back.
- **Crash points.** Each of the policies' six crash points ([landing.md](./landing.md#crash-points)) is reached by a
  `blizzard:crash-sweep` scenario, and the chunk lands exactly once with its markers whole.
- **Invariants.** `hub:landing-position-on-link`, `hub:submission-terms-standing`, and the widened
  `hub:repository-references-in-tenant` pass on the carried-over store and after every scenario above.
