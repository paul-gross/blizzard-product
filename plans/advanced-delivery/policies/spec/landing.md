# Landing by policy

## `land` runs policies

The `land` action stops being one policy. It becomes a runner of them: each pass, for each repository its visit resolved
and has not yet landed, it asks that repository's policy where the repository stands, then combines the answers into the
pass's one answer. The package is `hub/delivery/actions/landing/`:

- `policy.py` — `ILandingPolicy`, with `requires`, `uses`, and `step(repo: RepoLanding) -> RepoState`, and `POLICIES`,
  keyed by `LandingPolicy`.
- `merge_on_checks.py` — the `land` action's stages as the forges epic left them, with its one costly lookup made cheap
  ([The cost of a pass](./budget.md#the-cost-of-a-pass)).
- `fast_forward.py` and `park.py` — the two new policies.
- `checks.py` and `head_gate.py` — `Verdict`, its inherited-failure rule and one-time re-run, and `gate_head` with
  `base_merge.py`, moved out of the merge-on-checks stages so every policy reads checks and gates heads the same way.
- `combine.py` — the order and the one answer ([Combining](#combining-a-chunks-repositories)).

`RepoLanding` is one repository's slice of the visit: its `ResolvedRepository`, its submitted commit, its
`RepositoryForge` (`refs`, `changes`, `checks`, `offered`), and the markers its node holds so far, all handed in by the
visit. Like every action, a policy writes nothing itself: each marker below is handed to the pass's `effects`, and the
driver makes it durable before the pass goes on. `RepoState` is one of:

| State                       | Means                                                                                              |
| --------------------------- | -------------------------------------------------------------------------------------------------- |
| `Landed(revision)`          | The base holds the repository's work; `merged/<repo>` records `revision`                           |
| `Ready(land)`               | Proven and landable; `land` is the one forge write that lands it, taken when the order allows      |
| `Waiting(reason)`           | Checks running, a forge still deciding, a mechanical update in flight                              |
| `AwaitingMerge(change)`     | A parked change, open, waiting on a person                                                         |
| `Conflict(detail)`          | The base's tip cannot be brought in mechanically — a textual conflict, or a forge that cannot try  |
| `Failure(findings)`         | A check the repository's own work fails, or a foreign head                                         |
| `Inherited(findings)`       | Only checks the base also fails, still red after their one re-run                                  |
| `NeedsPerson(cause, about)` | Something no agent can fix: a parked change closed unmerged, or a base its forge will not let move |

Every policy hands on the markers `land` writes today where it does the same thing — `delivery-pr/<repo>/<n>` as soon as
a change is known, `merged/<repo>` as soon as a repository lands, `delivery-findings` on a failure or a substantive
wait, `ci-rerun/<…>` per re-run — so the board, the bounce envelope, and the prompts that read them are unchanged.

## The three policies

**`merge-on-checks`** is what every repository does today: find or open the change, gate its head, bring it up to its
base when it is behind, read its checks, and once every repository is ready merge it with `changes.merge.merge` and the
expected head. Its `Ready.land` is that merge, and the revision it lands at is the change's `merged_revision`. It reads
a change's checks through `change_checks(number)`, which the GitHub adapter answers from the head revision's check runs
([seam.md](./seam.md#the-seam-grows)). Whether it has already landed is one filtered read,
`changes(closed, base, head=<the chunk's branch>)`, in place of the ten pages of closed pull requests `land_pr_ci` scans
today.

**`fast-forward`** lands the chunk's revision with no change at all, and never rewrites the base's own history:

1. It reads the base's tip and compares it with the chunk's revision — the submitted commit, or a head the gate admits
   past it. A base that already holds the revision is `Landed` at the base's tip, as an empty landing is today.
2. A revision strictly ahead of the base is the candidate as it stands.
3. A base that has moved on since the chunk's branch left it is brought in mechanically, base first. With `refs.create`,
   `branches.merge`, and `refs.delete`, the policy creates a landing branch at the base's tip, named for the chunk's
   branch with `-landing` appended, and merges the chunk's revision into it with `merge_into`, whose message is the
   merge message `land` writes today, the feature's title and each work item's line. The merge commit's first parent is
   the base's tip and its second is the chunk's work, so `git log --first-parent` on the base still walks the base's own
   history once the base advances to it. That merge commit is the candidate. A merge the forge refuses as a conflict is
   `Conflict`, and the landing branch is deleted. Without the three, a diverged branch is `Conflict` at once, and the
   graph's conflict edge brings the base in as it does today. A landing branch the pass finds already holding the
   chunk's revision on top of the base's current tip is reused rather than rebuilt.
4. The candidate is proven by its own checks, read as `merge-on-checks` reads a change's: the same green, the same own
   and inherited failures, the same one-time re-run. A fast-forwarding repository's CI must run on pushes to the chunk's
   branches and their `-landing` branches, since there is no change to run it on; a candidate with no check at all stays
   `Waiting`, and the node's poll timeout bounces it as any long wait.
5. Proven, it is `Ready`, and its `land` is
   `advance_ref(base, expected=<the tip it compared against>,
   revision=<the candidate>)`. The advance is
   compare-and-swap: a base that moved in between refuses it, the pass answers `pending`, and the next pass builds the
   candidate again from the new tip. The landed revision is the proven candidate itself, and its landing branch is
   deleted once `merged/<repo>` is durable. A base its forge refuses to advance because only a pull request may change
   it is `NeedsPerson(base-protected)`.

A pure fast-forward makes no merge commit, so the work items' lines ride only on a landing that needed one; a
fast-forwarding repository whose shop wants every landing to carry them asks it of its chunks' commits.

**`park`** opens the change and waits for a person:

1. It finds or opens the change as `merge-on-checks` does, hands on `delivery-pr/<repo>/<n>`, and in the same epoch
   `awaiting-external-merge`, the marker the board already reads as "Awaiting your merge"
   (`hub/domain/chunk/delivery_read.py`). It reads no checks and gates no head: what merges is the person's call, and
   they may push to the branch before they do.
2. An open change is `AwaitingMerge`. A merged one is `Landed` at its `merged_revision`; a head that advanced past the
   submitted commit hands on a `delivery-findings` note naming the commits the person added, for the record, not as a
   failure.
3. A change closed without merging is `NeedsPerson(change-closed)`: a person said no, and an agent cannot argue with
   them. A visit in a later epoch does not count a change closed before the epoch began as its own; it finds no open
   change and opens a fresh one, so requeuing the chunk asks again and stopping it lets the work go.
4. It is never `Ready`: the hub does not merge a parked change, so it has no `land` of its own.

A parked change is noticed merged by polling alone. Rather than reading each parked change on every poll, `park` asks
`ForgeRegistry.recently_closed(repository)`, one listing of the changes into the base most recently updated — GitHub's
`state=closed&sort=updated&direction=desc` — shared by every parked change on the repository and cached for the park
poll floor ([Awaiting a merge](#awaiting-a-merge)). Ordering by update keeps a change parked for a week, then merged, at
the top of the listing however busy the repository. A change absent from it is still open; a change that might have
fallen off a full page is read alone with `change(n)`.

## What each policy proves

The tree that reaches the base is the tree that was verified only under `fast-forward`, where the base advances by
compare-and-swap to exactly the candidate its checks passed on. `merge-on-checks` proves the change: its checks ran on
its head, brought up to its base when the forge called it behind, but a base that moves after the checks pass is merged
in by the forge without a further run, unless the repository's forge itself requires a change to be up to date before it
merges — GitHub's "require branches to be up to date" — which closes that gap on the forge's side. `park` lands whatever
a person merged, and proves nothing on their behalf. The order a chunk's repositories land in does not prove their
combination either: a consumer waiting behind its parked library lands on checks that ran against the library as it was.
Proving the combination whatever the policy is the `trains` epic's.

## Combining a chunk's repositories

`combine.py` sorts a visit's repositories by `ResolvedRepository.landing_position` — lower first, unranked last, ties by
slug ([record.md](./record.md#the-order-a-chunks-repositories-land-in)) — and folds their states into one answer:

1. **A person's call wins.** Any `NeedsPerson` answers the pass with an escalation naming its cause and what it is about
   ([What goes to a person](#what-goes-to-a-person)). Nothing lands.
2. **Then a verdict.** Any `Conflict`, `Failure`, or `Inherited` gives the pass `conflict`, `failure`, or
   `inherited-failure`, with the priority `land_pr_ci` gives them today. Its findings are handed on, and nothing lands.
3. **Nothing lands until the hub's own repositories are ready.** While any repository that is not parked is `Waiting`,
   the pass answers `pending`. This is the chunk atomicity `land_pr_ci` has today, kept for every policy the hub lands
   itself.
4. **Then one at a time, in order.** Once every repository that is not parked is prepared — `Ready` or `Landed` here,
   joined by any state a later epic counts as prepared — the pass walks the order and takes each `Ready` repository's
   `land` once every repository before it has landed, handing on `merged/<repo>` after each. A `land` that fails as a
   stale write answers `pending`, and those already landed stay landed.
5. **A parked change waits its turn.** A parked repository opens its change only once every repository before it in the
   order has landed, so a person is never asked to merge a consumer before its library is in. Every repository after a
   parked one is `Ready` but waits for the person's merge.
6. **The answer.** `landed` once every repository has landed. `awaiting-merge` when everything still unlanded is a
   parked change or waits behind one. `pending` otherwise.

Landing across repositories, and across forges, is never one atomic act. What the hub promises is that it lands none of
the repositories it lands itself until all of them are ready, then lands them in order. A repository refused after
another has landed leaves the landed one standing, with its marker, its `delivery_repo_landed` row, and its flow's
`landed` fact, and the chunk reconciles as it does today: `_route` bounces it with the unlanded list, and a later visit
lands only what is left.

## What goes to a person

Three of a landing's outcomes are not an agent's to fix, so `land` answers them with an escalation rather than a
verdict, and the driver records it with its cause in `EscalationCause` (`foundation/escalation_causes.py`), beside the
actions epic's `unroutable-outcome` and the forges epic's `forge-auth`, `forge-config`, and `forge-capability-missing`:

| Cause            | Raised when                                                                     | A person resolves it by                                      |
| ---------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `change-closed`  | A parked change was closed without merging                                      | Requeuing, which opens a fresh change, or stopping the chunk |
| `base-protected` | The forge refused to advance a fast-forwarding base that only a change may move | Changing the policy or the protection, then requeuing        |
| `park-timeout`   | A parked change waited past its repository's bound ([below](#awaiting-a-merge)) | Merging or closing the change, then requeuing; or stopping   |

`EscalationCause` gains `CHANGE_CLOSED`, `BASE_PROTECTED`, and `PARK_TIMEOUT` for them.

An escalation is not a verdict. The chunk becomes `needs_human` and the escalation opens the node's next epoch, as every
hub-node escalation does; its submission keeps its place on its flows and is passed over while it waits. Requeuing it
releases its runner's lease as requeue does today and supersedes that submission, and the flows epic's reconcile submits
the chunk again at the back of each line. Stopping it withdraws the submission as any stop does.

## Awaiting a merge

`awaiting-merge` is reserved beside `pending` (`HUB_AWAITING_MERGE_CHOICE` in `hub/domain/graph/model.py`): never
authored, and never a verdict. Neither reserved word is refused at mint today — `HUB_PENDING_CHOICE` is kept out of
graphs by review alone — so `validation.py` gains the refusal for both: a node authoring a choice named `pending` or
`awaiting-merge` is refused. A pass that answers `awaiting-merge` ends its turn with that outcome; its submission keeps
its place on its flows and is ineligible until its next poll is due, exactly as after `pending`
([flows' turns](../../flows/spec/turns.md#pending)).

It differs from `pending` in its clock. `hub_node_poll` gains `kind`, a string column naming the wait the row records —
`pending` or, from this epic, `awaiting-merge`, with further wait kinds joining it as later epics need them — written by
the driver as it records the pass's answer, and the flows epic's eligibility reads the newest row's `kind` to know which
interval applies:

- **Its poll is slower.** The next poll is due after the node's `poll_interval` or `PARK_POLL_FLOOR`, two minutes,
  whichever is longer, so a parked change is read at most every two minutes however eager its graph.
- **Its wait is not a poll running long.** The node's `poll_timeout` measures only a run of `pending` rows: an
  `awaiting-merge` row ends the run, so a parked chunk is never bounced for waiting on a person.
- **Its wait is bounded by its repository.** From the first `awaiting-merge` row of the visit's current submission, the
  shortest `park_timeout_hours` among its unmerged parked repositories (168 when unset) bounds the wait. Past it, the
  pass answers an escalation with `park-timeout`, naming each unmerged change, so the chunk is `needs_human` and its
  runner's environments are a person's decision rather than a forgotten review's. The change stays open, and the clock
  starts again from the first wait of the submission the requeue leads to.

A parked chunk holds its runner's lease while it waits, as a `delivering` chunk does, and so counts against the runner's
agents until it lands, is escalated and requeued, or is stopped. The park timeout is the bound on that.

The chunk is `delivering` throughout, and its card shows the change and "Awaiting your merge" from the markers above.
Its places on its flows read `awaiting-merge` ([surfaces.md](./surfaces.md#the-board-and-the-flow-lanes)). A person
asking for changes on a parked change rather than merging it is not something this epic listens for: the change waits,
and the park timeout is what brings it back to a person's attention.

## Crash points

Each window between a forge write and the marker that records it is a named point in the crash-point registry
(`bzh:crash-point-registry`), each reached by a `blizzard:crash-sweep` scenario that kills the hub there and proves the
next pass converges:

| Point                                    | Between                                                               | The next pass                                               |
| ---------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| `policy.after-merge-into.before-advance` | The landing branch holds the base-first merge; the base has not moved | Finds the landing branch already holding it, and proves it  |
| `policy.after-advance.before-marker`     | The base advanced; `merged/<repo>` is not durable                     | Finds the base holding the candidate, and is `Landed` at it |
| `policy.after-merged-read.before-marker` | A parked change was read merged; `merged/<repo>` is not durable       | Reads it merged again, and records it                       |
