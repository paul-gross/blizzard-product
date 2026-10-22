# Landing by policy

## `land` runs policies

The `land` action stops being one policy. It becomes a runner of them: each pass, for each repository its visit resolved
and has not yet landed, it asks that repository's policy where the repository stands, then combines the answers into the
pass's one answer. The package is `hub/delivery/actions/landing/`:

- `policy.py` — `ILandingPolicy`, with one operation, `step(repo: RepoLanding) -> RepoState`, and `POLICIES`, keyed by
  `LandingPolicy`. What each policy requires of the forge, and what it uses when offered, is not the policy's to say: it
  is `POLICY_CAPABILITIES` in `hub/domain/config/landing_policies.py` ([record.md](./record.md#capabilities)). The
  package reads that module directly — `LandingPolicy`, `POLICY_CAPABILITIES`, and `DEFAULT_PARK_TIMEOUT_HOURS` — so the
  actions epic's purity allowlist (`test_hub_actions_import_only_their_allowlist`) gains
  `blizzard.hub.domain.config.landing_policies`, a module of constants that imports nothing outside `config`.
- `merge_on_checks.py` — the `land` action's stages as the forges epic left them, with its one costly lookup made cheap
  ([The cost of a pass](./budget.md#the-cost-of-a-pass)).
- `fast_forward.py` and `park.py` — the two new policies.
- `checks.py` and `head_gate.py` — `Verdict`, its inherited-failure rule and one-time re-run, and `gate_head` with
  `base_merge.py`, moved out of the merge-on-checks stages so every policy reads checks and gates heads the same way.
- `combine.py` — the order and the one answer ([Combining](#combining-a-chunks-repositories)).

`RepoLanding` is one repository's slice of the visit: its `ResolvedRepository`, its submitted commit, its
`RepositoryForge` (`refs`, `changes`, `checks`, `offered`), the markers its node holds so far, each with the time it was
written, and its landing terms — the policy, park timeout, and landing position the visit took when its delivery started
([record.md](./record.md#when-a-visit-takes-its-terms)). All of it is handed in by the visit. Like every action, a
policy writes nothing itself: each marker below is handed to the pass's `effects`, and the driver makes it durable
before the pass goes on. `RepoState` is one of:

| State                       | Means                                                                                                 |
| --------------------------- | ----------------------------------------------------------------------------------------------------- |
| `Landed(revision)`          | The base holds the repository's work; `merged/<repo>` records `revision`                              |
| `Ready(land)`               | Proven and landable; `land` is the one forge write that lands it, taken when the order allows         |
| `Waiting(reason)`           | Checks running, a forge still deciding, a mechanical update in flight                                 |
| `Held`                      | A parked repository whose change is not yet to open, behind the repositories ahead of it or the hub's |
| `AwaitingMerge(change)`     | A parked change, open, waiting on a person                                                            |
| `Conflict(detail)`          | The base's tip cannot be brought in mechanically — a textual conflict, or a forge that cannot try     |
| `Failure(findings)`         | A check the repository's own work fails, or a foreign head                                            |
| `Inherited(findings)`       | Only checks the base also fails, still red after their one re-run                                     |
| `NeedsPerson(cause, about)` | Something no agent can fix: a parked change closed unmerged, or a branch its forge will not let move  |

Every policy hands on the markers `land` writes today where it does the same thing — `delivery-pr/<repo>/<n>` as soon as
a change is known, `merged/<repo>` as soon as a repository lands (as the actions epic's admitted `LandingRecord`
effect), `delivery-findings` on a failure or a substantive wait, `ci-rerun/<…>` per re-run — so the board, the bounce
envelope, and the prompts that read them are unchanged.

## What a visit is handed for its policies

Three of the questions a policy asks are about time, and an action reads no clock and no store, so the driver reads each
into the `Visit` (`hub/domain/actions/visit.py`) before the pass, as the actions epic's purity rule asks of any input an
action needs:

- **When each marker was written.** The markers a `Visit` carries each carry their `produced_at`, which the store
  already keeps on every artifact row (`artifacts.produced_at`), so `park` can say how long its change has waited
  ([Awaiting a merge](#awaiting-a-merge)).
- **`Visit.first_turn_at`** — when the visit's first turn started: the earliest `started_at` among the `hub_turns` rows
  of every submission of the visit's `(chunk, node, epoch)`, whatever its `generation`, so a visit the flows epic
  resubmits because its resources changed keeps the moment it first began landing. A requeue, a restart, or a migration
  onto the node starts a new visit at a new epoch, so this is the moment a person last sent the chunk to land, never the
  escalation that stopped it.
- **`Visit.now`** — the driver's clock as it builds the visit, so a test steps it and no action calls a clock of its
  own.

The visit's landing terms reach each `RepoLanding` the same way, read from the submission
([record.md](./record.md#when-a-visit-takes-its-terms)).

## The three policies

**`merge-on-checks`** is what every repository does today: find or open the change, gate its head, bring it up to its
base when it is behind, read its checks, and once the order allows merge it with `changes.merge.merge` and the expected
head. Its `Ready.land` is that merge, and the revision it lands at is the change's `merged_revision`. It reads a
change's checks through `change_checks(number)`, which the GitHub adapter answers from the head revision's check runs
([seam.md](./seam.md#the-seam-grows)). Whether it has already landed is one filtered read,
`changes(closed, base, head=<the chunk's branch>)`, in place of the closed pull requests `land_pr_ci` pages through
today, up to ten pages of a hundred on a repository busy enough to have them. Green needs at least one check: a change
whose head carries no check at all stays `Waiting`, as it does today (`Verdict.green` in `land_pr_ci.py`), and the
node's poll timeout bounces it, so a repository without CI is not one to land by this policy. It merges with the `merge`
method only, as today. A repository whose forge admits only squash or rebase merges refuses that merge — GitHub's 405
"Merge commits are not allowed on this repository", which the adapter answers as `ForgeRejected("method")` — and no
agent can change a repository's settings, so the refusal is `NeedsPerson(merge-method-refused)` rather than a wait that
bounces three hours later. Choosing a method per repository waits for a shop that needs it.

**`fast-forward`** lands the chunk's revision with no change at all, and never rewrites the base's own history:

1. It reads the base's tip and compares it with the chunk's revision — the submitted commit, or a head the gate admits
   past it. A base that already holds the work is `Landed` at the revision that carries it, never at the base's tip,
   which by then may carry other chunks' work too: at the tip of the chunk's landing branch when that branch still
   exists and the base holds it, and otherwise at the chunk's revision. This differs from `merge-on-checks`, whose empty
   landing records the base's tip as today; a fast-forwarding base moves only to revisions the hub chose, so the
   revision that carries the chunk's work is always one it can name.
2. A revision strictly ahead of the base is the candidate as it stands.
3. A base that has moved on since the chunk's branch left it is brought in mechanically, base first. With `refs.create`,
   `branches.merge`, and `refs.delete`, the policy creates a landing branch at the base's tip, named for the chunk's
   branch with `-landing` appended, and merges the chunk's revision into it with `merge_into`, whose message is the
   merge message `land` writes today, the feature's title and each work item's line. The merge commit's first parent is
   the base's tip and its second is the chunk's work, so `git log --first-parent` on the base still walks the base's own
   history once the base advances to it. That merge commit is the candidate, a revision no check has seen yet, so its
   checks run afresh. A merge the forge refuses as a conflict is `Conflict`, and the landing branch is deleted. Without
   the three, a diverged branch is `Conflict` at once, and the graph's conflict edge brings the base in as it does
   today.
4. A landing branch the pass finds is reused only when its tip is a merge whose first parent is the base's current tip
   and whose second is the chunk's current revision. One found on an older tip — the base moved while its candidate was
   proving — or holding an older revision of the chunk — a conflict the chunk resolved since moved its branch — is
   deleted and built again, so the candidate always proves the tip it will advance and the work it will land.
5. The candidate is proven by its own checks, read as `merge-on-checks` reads a change's: the same green, needing at
   least one check, the same own and inherited failures, the same one-time re-run. A fast-forwarding repository's CI
   must run on pushes to the chunk's branches and their `-landing` branches, since there is no change to run it on; a
   candidate with no check at all stays `Waiting`, and the node's poll timeout bounces it as any long wait.
6. Proven, it is `Ready`, and its `land` is `advance_ref(base, expected, candidate)`, where `expected` is the tip it
   compared against. The advance is fast-forward-only, and the candidate was built on that tip, so a base that moved in
   between refuses it: the pass answers `pending`, and the next pass builds the candidate again from the new tip. The
   landed revision is the proven candidate itself, and its landing branch is deleted once `merged/<repo>` is durable. A
   base its forge refuses to advance because a rule of the repository forbids it — a protection, a ruleset, or a
   required check that only runs on pull requests — is `NeedsPerson(branch-protected)`.

A busy base can move again every time a candidate is proven, and each move costs a rebuild and a fresh CI run. Nothing
counts the rebuilds: a candidate that never catches its base waits as any long wait does, inside the node's poll
timeout, and the bounce at its end is what tells someone the repository moves faster than its CI. A chunk landing into
several repositories feels this most, since it advances none of them until all are ready.

A landing branch is deleted when its repository lands, when its merge conflicts, and when a later pass finds it on a
stale tip. A chunk stopped, escalated, or restarted onto an earlier node leaves its landing branch behind; the page on
choosing a policy (`docs/deployment/`) names the `-landing` suffix so an operator can prune what is left, and sweeping
them is not this epic's.

A pure fast-forward makes no merge commit, so the work items' lines ride only on a landing that needed one; a
fast-forwarding repository whose shop wants every landing to carry them asks it of its chunks' commits.

**`park`** opens the change and waits for a person:

1. Until the order allows it to open ([Combining](#combining-a-chunks-repositories)), a parked repository is `Held`, and
   `park` makes no forge call for it.
2. Allowed, it finds or opens the change as `merge-on-checks` does, hands on `delivery-pr/<repo>/<n>`, and in the same
   epoch `awaiting-external-merge`, the marker the board already reads as "Awaiting your merge"
   (`hub/domain/chunk/delivery_read.py`). It reads no checks and gates no head: what merges is the person's call, and
   they may push to the branch before they do.
3. The change it watches is the one its newest `delivery-pr/<repo>/<n>` marker names, across every epoch of the visit's
   node. An open change is `AwaitingMerge` — from the visit's first pass, whatever rule 1 would say, since a change an
   earlier visit opened was opened only once everything ahead of it had landed, and what landed stays landed. The board
   reads "Awaiting your merge" only where `awaiting-external-merge` shares an epoch with the change's newest
   `delivery-pr` marker, so a visit that adopts an open change from an earlier epoch hands on both markers again in its
   own epoch, on the first pass that finds them missing there. A merged change is `Landed` at its `merged_revision`,
   whenever the person merged it — a change merged while its chunk sat escalated lands the requeued visit at once; a
   head that advanced past the submitted commit hands on a `delivery-findings` note naming the commits the person added,
   for the record, not as a failure.
4. A change closed without merging is `NeedsPerson(change-closed)`: a person said no, and an agent cannot argue with
   them. A change closed before `Visit.first_turn_at` is not the visit's own: the visit finds no open change and opens a
   fresh one, so a person who closes the change while the chunk sits escalated and then requeues it is asked again, and
   stopping the chunk lets the work go.
5. It is never `Ready`: the hub does not merge a parked change, so it has no `land` of its own.

A parked change is noticed merged by polling alone. Rather than reading each parked change on every poll, `park` asks
the repository's `IForgeChanges.recently_closed(base, limit=100)` ([seam.md](./seam.md#the-seam-grows)), one listing of
the changes into the base most recently updated. Every parked change on a repository asks for the same listing at the
same address, so the adapter's conditional read answers all but the first poll after something closed with a free `304`
([budget.md](./budget.md#the-cost-of-a-pass)); the listing itself is never kept between polls, so a merge is noticed at
the first poll after it. A change in the listing is merged or closed as it says. A change absent from a listing shorter
than its limit is still open; a change absent from a full one might have fallen off it and is read alone with
`change(n)`.

What a forge answers to the operations this epic adds decides the repository's state, beside the forges epic's table for
the operations `land` already spoke ([forges' landing](../../forges/spec/landing.md#what-the-forge-answers)), whose two
rules for every operation — a rejected credential and an unknown repository go to a person — hold here too:

| Operation                        | Answer                                                                      | State                                                                                              |
| -------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `advance_ref`                    | `ForgeRejected("stale")`                                                    | `Waiting`; the pass answers `pending`, and the next builds the candidate again on the new tip      |
| `advance_ref`                    | `ForgeRejected("protected")`                                                | `NeedsPerson(branch-protected)`, naming the base and the forge's message                           |
| `advance_ref`                    | Any other `ForgeRejected`, or `ForgeNotFound("branch")`                     | `Waiting`; the next pass re-reads the tip, and a base that already holds the candidate is `Landed` |
| `create_ref`                     | `ForgeRejected("stale")`, a branch already there at another revision        | `Waiting`; the next pass finds the branch and reuses or rebuilds it (rule 4)                       |
| `create_ref`, `merge_into`       | `ForgeRejected("protected")`, a rule forbidding the hub's `-landing` branch | `NeedsPerson(branch-protected)`, naming the landing branch and the forge's message                 |
| `merge_into`                     | `ForgeRejected("conflict")`                                                 | `Conflict`; the landing branch is deleted                                                          |
| `create_ref`, `merge_into`       | Any other `ForgeRejected`, or `ForgeNotFound`                               | `Waiting`                                                                                          |
| `delete_ref`, after landing      | Any refusal                                                                 | `Landed` stands; the branch is left for an operator to prune                                       |
| `delete_ref`, before a rebuild   | Any refusal                                                                 | `Waiting`; the next pass tries again                                                               |
| `recently_closed`                | Any `ForgeRejected`                                                         | `AwaitingMerge` stands; the change is read again at the next poll                                  |
| `change_checks`                  | `ForgeNotFound("revision")`, or any `ForgeRejected`                         | `Waiting`, as a degraded check read waits today                                                    |
| `changes.merge`, merge-on-checks | `ForgeRejected("method")`                                                   | `NeedsPerson(merge-method-refused)`, naming the repository                                         |

A `Waiting` repository that never stops waiting is bounded by its node's poll timeout, as every wait is.

## What each policy proves

The tree that reaches the base is the tree that was verified only under `fast-forward`, where the base advances,
fast-forward-only, to exactly the candidate its checks passed on. `merge-on-checks` proves the change: its checks ran on
its head, brought up to its base when the forge called it behind, but a base that moves after the checks pass is merged
in by the forge without a further run, unless the repository's forge itself requires a change to be up to date before it
merges — GitHub's "require branches to be up to date" — which closes that gap on the forge's side. `park` lands whatever
a person merged, and proves nothing on their behalf. The order a chunk's repositories land in does not prove their
combination either: a consumer waiting behind its parked library lands on checks that ran against the library as it was.
Proving the combination whatever the policy is the `trains` epic's.

## Combining a chunk's repositories

`combine.py` sorts a visit's repositories by their terms' `landing_position` — lower first, unranked last, ties by slug
([record.md](./record.md#the-order-a-chunks-repositories-land-in)) — and folds their states into one answer. A
repository the hub lands itself is one whose policy is `fast-forward` or `merge-on-checks`; it is **prepared** when it
is `Ready` or `Landed`, or in any state a later epic counts as prepared.

1. **While a person reviews, only their change is read.** When the first unlanded repository in the order is a parked
   one whose change is open, it is the only repository the pass steps. Every repository ahead of it has landed — its
   change opened only once they had — and every one after it waits for the person's merge whatever state it was last in,
   so the pass makes no forge call for them: no update from a moved base, no rebuilt candidate, no fresh CI run, for as
   long as the review takes. Its state alone answers the pass: `AwaitingMerge` answers `awaiting-merge`, `NeedsPerson`
   an escalation, and `Landed` hands on `merged/<repo>` and lets the pass go on to the rules below with that repository
   landed, so the repositories after it are stepped afresh — brought up to a base that moved while the person reviewed,
   proven again — before any of them lands. This rule comes first: while a parked change is open, a `Conflict` or
   `Failure` a later repository showed before the change opened is not read, and so never decides the pass.
2. **A person's call wins.** Otherwise, any `NeedsPerson` answers the pass with an escalation naming its cause and what
   it is about ([What goes to a person](#what-goes-to-a-person)). Nothing lands.
3. **Then a verdict.** Any `Conflict`, `Failure`, or `Inherited` gives the pass `conflict`, `failure`, or
   `inherited-failure`, with the priority `land_pr_ci` gives them today. Its findings are handed on, and nothing further
   lands.
4. **The hub's own are proven first.** Every repository the hub lands itself and has not yet landed is stepped each
   pass. While any of them is not yet prepared, the pass answers `pending` and lands nothing. This is the chunk
   atomicity `land_pr_ci` has today, kept for every policy the hub lands itself.
5. **A parked change opens last of all that must be ready.** A parked repository whose change is not yet open is stepped
   — its change found or opened — only once every repository ahead of it in the order has landed and every repository
   the hub lands itself is prepared; until then it is `Held`. A person is therefore asked to review only when the rest
   of the chunk is proven, and never to merge a consumer before its library is in.
6. **Then one at a time, in order.** Once every repository the hub lands itself is prepared, the pass walks the order
   and takes each `Ready` repository's `land` once every repository before it has landed, handing on `merged/<repo>`
   after each, and stops at the first parked repository not yet `Landed`; that repository's change opens in this pass,
   and from the next pass rule 1 holds. A `land` refused as stale answers `pending`, and those already landed stay
   landed.
7. **The answer.** `landed` once every repository has landed; `awaiting-merge` while a parked change is open; `pending`
   otherwise.

Landing across repositories, and across forges, is never one atomic act. What the hub promises is that it lands none of
the repositories it lands itself until all of them are ready, then lands them in order. A repository refused after
another has landed leaves the landed one standing, with its marker, its `delivery_repo_landed` row, and its flow's
`landed` fact, and the chunk reconciles as it does today: `_route` bounces it with the unlanded list, and a later visit
lands only what is left.

## What goes to a person

Four of a landing's outcomes are not an agent's to fix, so `land` answers them with an escalation rather than a verdict,
and the driver records it with its cause in `EscalationCause` (`foundation/escalation_causes.py`), beside the actions
epic's `UNROUTABLE_OUTCOME` and `HUB_ACTION_MISSING` and the forges epic's `forge-auth`, `forge-config`, and
`forge-capability-missing`:

| Cause                  | Raised when                                                                                        | A person resolves it by                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `change-closed`        | A parked change was closed without merging                                                         | Requeuing, which opens a fresh change, or stopping the chunk  |
| `branch-protected`     | The forge refused to move a fast-forwarding base, or the hub's `-landing` branch, outside a change | Changing the policy or the repository's rule, then requeuing  |
| `merge-method-refused` | The forge refused a `merge-on-checks` merge because the repository admits no merge commits         | Allowing merge commits or changing the policy, then requeuing |
| `park-timeout`         | A parked change waited past its repository's bound ([below](#awaiting-a-merge))                    | Merging or closing the change, then requeuing; or stopping    |

`EscalationCause` gains `CHANGE_CLOSED`, `BRANCH_PROTECTED`, `MERGE_METHOD_REFUSED`, and `PARK_TIMEOUT` for them.

An escalation is not a verdict. The driver writes each of them as it writes every escalation that ends a turn: through
the locked hub-lease escalation writer, at the visit's epoch plus one, recorded once per `(chunk, node, epoch)`
([flows' escalation](../../flows/spec/verdict.md#escalation)). The chunk becomes `needs_human` and the escalation opens
the node's next epoch; its submission keeps its place on its flows and is passed over while it waits. Requeuing it
releases its runner's lease as requeue does today and supersedes that submission, and the flows epic's reconcile submits
the chunk again at the back of each line, where its first turn takes fresh terms. Stopping it withdraws the submission
as any stop does.

## Awaiting a merge

`awaiting-merge` joins `pending` in the actions epic's `RESERVED_WAITS` (`hub/domain/graph/actions.py`), so it is never
authored and never a verdict: the actions epic already refuses a node authoring a reserved word, and exempts every
reserved word from the rule that a node routes each of its action's answers, so the shipped graphs, which author
neither, mint unchanged. `land`'s `answers` gain it. `Pass.next_poll_at` applies to it as to any reserved wait. A pass
that answers `awaiting-merge` ends its turn with that outcome; its submission keeps its place on its flows and is
ineligible until its next poll is due, exactly as after `pending` ([flows' turns](../../flows/spec/turns.md#pending)).

It differs from `pending` in its clock. `hub_node_poll` (`hub/store/schema.py`) gains `kind`, a string column naming the
wait the row records — `pending` or, from this epic, `awaiting-merge`, with further wait kinds joining it as later epics
need them — `NOT NULL` with a server default of `pending` ([carry-over.md](./carry-over.md#the-migrations)), and
`last_polled_at`, a nullable time. Both are written by the flows epic's `HubNodeRouter.record_wait(kind)`
(`hub/delivery/hub_node.py`) as it records the pass's answer. A `pending` pass adds a row, as today; it is bounded by
the poll timeout, so a visit adds at most a few hundred. An `awaiting-merge` pass is not bounded so tightly — a week's
review polled every two minutes would add some five thousand rows, every one of them loaded with the chunk
(`ChunkFacts.hub_node_polls`) — so a run of them is coalesced into one row: when the visit's newest row is already
`awaiting-merge`, the pass moves that row's `last_polled_at` forward, and replaces the row's `next_poll_at` with the
hint this pass answered, or none, rather than adding a row; the row's `polled_at` stays the moment the run began. The
two writes are one statement each, and a pass that dies between its answer and its write simply polls again.
`HubNodePollFact` carries the kind and `last_polled_at`, and the readers follow them:

- **Its poll is slower.** The flows epic's eligibility asks `PollPolicy.due_at(row)` (`hub/domain/delivery/flows.py`) of
  the visit's newest row, and this epic changes only the interval it starts from: `due_at` measures
  `PollPolicy.interval_for(row.kind)` from the row's `last_polled_at`, or its `polled_at` where it has none, and still
  takes the later of that and the row's `next_poll_at`, the actions epic's hint. For `pending`, `interval_for` answers
  the node's `poll_interval`, as before; for `awaiting-merge` it answers the node's `poll_interval` or
  `PARK_POLL_FLOOR`, two minutes, whichever is longer, so a parked change is read at most every two minutes however
  eager its graph. `PARK_POLL_FLOOR` lives in `hub/domain/config/landing_policies.py`.
- **Its wait is not a poll running long.** The node's `poll_timeout` measures only the newest unbroken run of `pending`
  rows: `ChunkFacts.hub_node_poll_history` (`hub/domain/chunk/model.py`) still answers every row, oldest first, and the
  timeout's start becomes the first row of that run rather than the first row of the visit, so an `awaiting-merge` row
  ends a run and a parked chunk is never bounced for waiting on a person. `standards/hub-nodes/outcome-protocol.md`'s
  "from the first recorded pending attempt" becomes "from the first pending attempt since the last wait of another
  kind".
- **The board says which wait it is.** `hub_node_pending()` (`ChunkFacts`, `hub/domain/chunk/model.py`) answers the
  newest row with its kind and when its run began, and `PendingView` carries both
  ([surfaces.md](./surfaces.md#the-board-and-the-flow-lanes)), so the chunk detail reads "awaiting merge since …" for a
  parked chunk rather than the line a chunk waiting on its checks shows.
- **Its wait is bounded by its repository.** Each parked change is measured on its own clock, against its own
  repository's terms: from the `produced_at` of its `delivery-pr/<repo>/<n>` marker in the visit's own epoch — written
  when the change opened, or again when the visit adopted it from an earlier epoch — to `Visit.now`, against that
  repository's `park_timeout_hours` (`DEFAULT_PARK_TIMEOUT_HOURS`, 168, in `hub/domain/config/landing_policies.py`, when
  unset). A chunk parked in two repositories opens the second change only after the first merges, so the second gets its
  whole bound, never what the first review left of a shared one. Past its bound, the pass answers an escalation with
  `park-timeout`, naming the change, so the chunk is `needs_human` and its runner's environments are a person's decision
  rather than a forgotten review's. The change stays open, and the requeued visit, adopting it, hands on its marker
  afresh in its own epoch, which starts the clock again.

A parked chunk holds its runner's lease while it waits, as a `delivering` chunk does, and so counts against the runner's
agents until it lands, is escalated and requeued, or is stopped. The park timeout is the bound on that.

The chunk is `delivering` throughout, and its card shows the change and "Awaiting your merge" from the markers above.
Its places on its flows read `awaiting-merge` ([surfaces.md](./surfaces.md#the-board-and-the-flow-lanes)). A person
asking for changes on a parked change rather than merging it is not something this epic listens for: the change waits,
and the park timeout is what brings it back to a person's attention.

## Crash points

Each window between a forge write and the next durable record is a named point in the crash-point registry
(`bzh:crash-point-registry`, `foundation/crash.py`), reached by a `blizzard:crash-sweep` scenario that kills the hub
there and proves the next pass converges. Six windows are the policies' own, beside the actions epic's general
`action.after-effect.before-write`:

| Point                                       | Between                                                                    | The next pass                                                                                                                                                                             |
| ------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `policy.after-create-ref.before-merge-into` | The landing branch exists at the base's tip; the chunk's work is not in it | Finds the landing branch at the tip, and merges the work into it                                                                                                                          |
| `policy.after-merge-into.before-advance`    | The landing branch holds the base-first merge; the base has not moved      | Finds the landing branch already holding it, and proves it                                                                                                                                |
| `policy.after-advance.before-marker`        | The base advanced; `merged/<repo>` is not durable                          | Finds the base holding the candidate, and is `Landed` at the candidate — the landing branch's tip, or the chunk's revision for a pure fast-forward — however far the base has moved since |
| `policy.after-marker.before-delete-ref`     | `merged/<repo>` is durable; the landing branch is not deleted              | Is `Landed` by its marker, and deletes the landing branch it finds                                                                                                                        |
| `policy.after-open-change.before-marker`    | A parked change is open; `delivery-pr/<repo>/<n>` is not durable           | Finds the open change from the chunk's branch, and adopts it                                                                                                                              |
| `policy.after-merged-read.before-marker`    | A parked change was read merged; `merged/<repo>` is not durable            | Reads it merged again, and records it                                                                                                                                                     |
