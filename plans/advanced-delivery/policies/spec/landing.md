# Landing by policy

## `land` runs policies

The `land` action stops being one policy. It becomes a runner of them: each pass, for each repository its visit resolved
and has not yet landed, it asks that repository's policy where the repository stands, then combines the answers into the
pass's one answer. The package is `hub/delivery/actions/landing/`:

- `policy.py` — `ILandingPolicy`, with `requires`, `uses`, and `step(repo: RepoLanding) -> RepoState`, and `POLICIES`,
  keyed by `LandingPolicy`.
- `merge_on_checks.py` — the `land` action's stages as the forges slice left them, unchanged.
- `fast_forward.py` and `park.py` — the two new policies.
- `checks.py` and `head_gate.py` — `Verdict`, its inherited-failure rule and one-time re-run, and `gate_head` with
  `base_merge.py`, moved out of the merge-on-checks stages so every policy reads checks and gates heads the same way.
- `combine.py` — the order and the one answer ([Combining](#combining-a-chunks-repositories)).

`RepoLanding` is one repository's slice of the visit: its `ResolvedRepository`, its submitted commit, its adapter from
`ForgeRegistry`, its markers so far, and the `MarkerWriter`. `RepoState` is one of:

| State                   | Means                                                                                         |
| ----------------------- | --------------------------------------------------------------------------------------------- |
| `Landed(revision)`      | The base holds the repository's work; `merged/<repo>` records `revision`                      |
| `Ready(land)`           | Proven and landable; `land` is the one forge write that lands it, taken when the order allows |
| `Waiting(reason)`       | Checks running, a forge still deciding, a mechanical update in flight                         |
| `AwaitingMerge(change)` | A parked change, open, waiting on a person                                                    |
| `Conflict(detail)`      | A textual conflict no mechanical update could settle                                          |
| `Failure(findings)`     | A check the repository's own work fails, a foreign head, or a parked change closed unmerged   |
| `Inherited(findings)`   | Only checks the base also fails, still red after their one re-run                             |

Every policy writes the markers `land` writes today where it does the same thing — `delivery-pr/<repo>/<n>` as soon as a
change is known, `merged/<repo>` as soon as a repository lands, `delivery-findings` on a failure or a substantive wait,
`ci-rerun/<…>` per re-run — so the board, the bounce envelope, and the prompts that read them are unchanged.

## The three policies

**`merge-on-checks`** is what every repository does today, moved without change: find or open the change, gate its head,
bring it up to its base when it is behind, read its checks, and once every repository is ready merge it with
`changes.merge.merge` and the expected head. Its `Ready.land` is that merge.

**`fast-forward`** lands the chunk's revision with no change at all:

1. It reads the base's tip and compares it with the chunk's revision — the submitted commit, or a head the gate admits
   past it. A base that already holds the revision is `Landed` at the base's tip, as an empty landing is today.
2. A base that has moved on since the chunk's branch left it is brought in mechanically: with `branches.merge`, the
   policy merges the base's tip into the chunk's branch, which the head gate admits as it admits the base merges
   `update-branch` makes today, and answers `Waiting` while the new head's checks run. A merge the forge refuses as a
   conflict is `Conflict`. Without `branches.merge`, a diverged branch is `Conflict` at once, and the graph's conflict
   edge brings the base in as it does today.
3. A revision ahead of the base is proven by its own checks, read as `merge-on-checks` reads a change's: the same green,
   the same own and inherited failures, the same one-time re-run. A fast-forwarding repository's CI must run on pushes
   to the chunk's branches, since there is no change to run it on; a revision with no check at all stays `Waiting`, and
   the node's poll timeout bounces it as any long wait.
4. Proven, it is `Ready`, and its `land` is
   `advance_ref(base, expected=<the tip it compared against>,
   revision=<the proven revision>)`. The advance is
   compare-and-swap: a base that moved in between refuses it, the pass answers `pending`, and the next pass compares
   again. The landed revision is the proven revision itself.

**`park`** opens the change and waits for a person:

1. It finds or opens the change as `merge-on-checks` does, writes `delivery-pr/<repo>/<n>`, and in the same epoch writes
   `awaiting-external-merge`, the marker the board already reads as "Awaiting your merge"
   (`hub/domain/chunk/delivery_read.py`). It reads no checks and gates no head: what merges is the person's call, and
   they may push to the branch before they do.
2. An open change is `AwaitingMerge`. A merged one is `Landed` at its merge revision; a head that advanced past the
   submitted commit writes a `delivery-findings` note naming the commits the person added, for the record, not as a
   failure. A change closed without merging is `Failure`, with findings naming the change and that it was closed.
3. It is never `Ready`: the hub does not merge a parked change, so it has no `land` of its own.

A parked change is noticed merged by polling alone. Rather than reading each parked change on every poll, `park` asks
`ForgeRegistry.recently_closed(repository)`, one listing of the changes closed into the base, newest first, shared by
every parked change on the repository and cached for the park poll floor ([Awaiting a merge](#awaiting-a-merge)). A
change absent from it is still open; a change that might have fallen off a full page is read alone with `change(n)`.

## Combining a chunk's repositories

`combine.py` orders a visit's repositories by their `lands_after`, keeping only the edges between repositories the visit
holds, and folds their states into one answer:

1. **A verdict wins.** Any `Conflict`, `Failure`, or `Inherited` gives the pass `conflict`, `failure`, or
   `inherited-failure`, with the priority `land_pr_ci` gives them today. Its findings are written, and nothing lands.
2. **Nothing lands until the hub's own repositories are ready.** While any repository that is not parked is `Waiting`,
   the pass answers `pending`. This is the chunk atomicity `land_pr_ci` has today, kept for every policy the hub lands
   itself.
3. **Then in order.** Once every repository that is not parked is `Ready` or `Landed`, the pass takes each `Ready`
   repository's `land` whose predecessors have all landed, predecessors first, recording `merged/<repo>` after each. A
   `land` that fails as a stale write answers `pending`, and those already landed stay landed.
4. **Parked changes wait their turn in the order.** A parked repository opens its change only once every repository it
   lands after has landed, so a person is never asked to merge a consumer before its library is in. A repository that
   lands after a parked one is `Ready` but waits for the person's merge.
5. **The answer.** `landed` once every repository has landed. `awaiting-merge` when everything still unlanded is a
   parked change or waits behind one. `pending` otherwise.

A repository refused after another has landed leaves the landed one standing, with its marker, its
`delivery_repo_landed` row, and its flow's `landed` fact, and the chunk reconciles as it does today: `_route` bounces it
with the unlanded list, and a later visit lands only what is left.

## Awaiting a merge

`awaiting-merge` is reserved beside `pending` (`HUB_AWAITING_MERGE_CHOICE` in `hub/domain/graph/model.py`): never
authored, refused as a choice at mint by `validation.py` as `pending` is, and never a verdict. A pass that answers it
ends its turn with that outcome; its submission keeps its place on its flows and is ineligible until its next poll is
due, exactly as after `pending` ([flows' turns](../../flows/spec/turns.md#pending)).

It differs from `pending` in its clock. `hub_node_poll` gains `kind`, `pending` or `awaiting-merge`, written by the
executor's `_record_pending` and a new `_record_awaiting`:

- **Its poll is slower.** The next poll is due after the node's `poll_interval` or `PARK_POLL_FLOOR`, two minutes,
  whichever is longer, so a parked change is read at most every two minutes however eager its graph.
- **Its wait is not a poll running long.** The node's `poll_timeout` measures only a run of `pending` rows: an
  `awaiting-merge` row ends the run, so a parked chunk is never bounced for waiting on a person.
- **Its wait is bounded by its repository.** From the first `awaiting-merge` row of the visit — or the resolution of the
  visit's last park escalation, whichever is newer — the shortest `park_timeout_hours` among its unmerged parked
  repositories (168 when unset) bounds the wait. Past it, the executor records an escalation with the new
  `EscalationCause.PARK_TIMEOUT` (`foundation/escalation_causes.py`), naming each unmerged change, so the chunk is
  `needs_human` and its runner's environments are a person's decision rather than a forgotten review's. The change stays
  open. Resolving the escalation returns the chunk to `delivering` and restarts the clock; stopping it withdraws the
  submission as any stop does.

The chunk is `delivering` throughout, and its card shows the change and "Awaiting your merge" from the markers above.
Its places on its flows read `awaiting-merge` ([surfaces.md](./surfaces.md#the-board-and-the-flow-lanes)).
