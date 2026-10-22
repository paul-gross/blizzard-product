# The seam the policies grow

## The seam grows

The forges epic split the seam by the caller that needs each part, and this epic grows each role only where a policy
needs it, every new operation behind its own capability. `IForgeRefs` (`hub/forge/landing.py`) gains the four
`fast-forward` needs:

| Operation                                 | Capability       | Answers                                                                                                                                                                                     |
| ----------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `advance_ref(branch, expected, revision)` | `refs.advance`   | Moves `branch` from `expected` to `revision`, a fast-forward; `ForgeRejected("stale")` when it was not at `expected`, `ForgeRejected("protected")` when a rule of the repository forbids it |
| `merge_into(branch, revision, message)`   | `branches.merge` | The merge commit on `branch` bringing `revision` in, `None` when `branch` already holds it, `ForgeRejected("conflict")` on a conflict, `ForgeRejected("protected")` when a rule forbids it  |
| `create_ref(branch, revision)`            | `refs.create`    | Creates `branch` at `revision`; an existing branch already at `revision` is not an error, one at another revision is `ForgeRejected("stale")`                                               |
| `delete_ref(branch)`                      | `refs.delete`    | Deletes `branch`; a branch already gone is not an error                                                                                                                                     |

The four are writes, so each goes out under the forges epic's per-credential write lock
([forges' failures and retries](../../forges/spec/seam.md#failures-and-retries)), one at a time per forge and secret,
like opening, updating, and merging a change. `ForgeRejected`'s reasons gain `method`, a merge refused because the
repository does not admit the method asked for; `stale`, `conflict`, and `protected`, which the forges epic named
without yet raising, are raised from here.

`IForgeChanges` gains `recently_closed(base, limit)` under `changes.read`: the changes into `base` that are closed or
merged, most recently updated first, at most `limit` of them. It is the one listing `park` reads to notice a person's
merge ([landing.md](./landing.md#the-three-policies)), and it is ordered by update rather than by creation, which is how
`changes(state, base, head)` orders, so that a change parked for a week and then merged sits at the top of it however
busy the repository has been since.

`IForgeChecks` gains `change_checks(number)` beside `checks(ref)`, under `checks.read`: the checks that decide whether a
change may merge. On GitHub those are the change's head's; a forge that runs a change's checks on something other than
its head answers from there, so `merge-on-checks` asks the change rather than guessing where its checks ran.
`update_from_base` answers the forges epic's `Update(outcome, method)`, where `method` says how the forge brought the
change up to its base — `merge` on GitHub — so the head gate knows what it is reading
([landing.md](./landing.md#the-three-policies)).

The four new `IForgeRefs` operations, `recently_closed`, and `change_checks` join `OPERATION_CAPABILITY` beside the
forges epic's, so `ForgeRegistry` refuses each where its capability is not offered. The four new capabilities join the
closed `Capability` set in `hub/domain/config/capabilities.py`, and GitHub's entry in `FORGE_KIND_CAPABILITIES`
(`hub/domain/config/forges.py`) gains them. A forge kind that cannot move a branch outside a change offers none of the
four, and its repositories cannot fast-forward ([record.md](./record.md#capabilities)). `IForgeRefs` reaches seven
operations, `IForgeChanges` six, and `IForgeChecks` three, each still under `bzh:seam-size-ceiling`'s twelve.

## The GitHub adapter

The GitHub adapter offers every new capability, each over GitHub's REST API, so the hub speaks one API to GitHub and
spends one rate bucket:

- **`advance_ref`** is `PATCH /repos/{o}/{r}/git/refs/heads/{branch}` with `force: false`. A non-forced update succeeds
  only when the new revision descends from the branch's tip, and that is the compare-and-swap a fast-forward needs: the
  policy builds its candidate on the tip it compared against, so a base that has since moved to anything the candidate
  does not descend from refuses the advance. The adapter answers a refusal by its message, matched by prefix, since
  GitHub and the mock both append detail: a 422 beginning "Update is not a fast forward" is `ForgeRejected("stale")`; a
  422 naming a protected branch or a repository rule — "Protected branch update failed", "Cannot update this protected
  ref", "Repository rule violations found", or a required status check the update lacks — is
  `ForgeRejected("protected")`. `expected` is what the adapter compares the branch's tip against before it sends the
  update, so a tip it already knows has moved is answered `stale` without a write.
- **`merge_into`** is `POST /repos/{o}/{r}/merges`, 201 a merge, 204 nothing to merge, 409 a conflict, and a 422 naming
  a protected branch or a repository rule `ForgeRejected("protected")` — a ruleset whose pattern covers the hub's
  `-landing` branches refuses here; it merges into the landing branch, so the merge commit's first parent is the branch
  it lands on.
- **`create_ref`** is `POST git/refs` and **`delete_ref`** `DELETE git/refs/heads/{b}`. A 422 "Reference already exists"
  is read back with `branch_tip`: at the same revision it is not an error, at another it is `ForgeRejected("stale")`. A
  422 or 404 on a branch already gone is not an error. A 422 naming a repository rule is `ForgeRejected("protected")`.
- **`recently_closed`** is `GET pulls?state=closed&base=<base>&sort=updated&direction=desc&per_page=<limit>`, one page.
  The adapter keeps no answer between calls: every parked change on a repository asks for the same address, and the
  conditional read answers it `304` for free until something closes ([budget.md](./budget.md#the-cost-of-a-pass)), so a
  merge is seen at the first poll after it.
- **`change_checks`** is `checks` of the change's head revision.
- **`changes.merge`**'s 405 "Merge commits are not allowed on this repository" is `ForgeRejected("method")`, where the
  forges epic answered every 405 with its message.

A base protected so that only pull requests may change it refuses the advance as `ForgeRejected("protected")`, which
`fast-forward` answers as `NeedsPerson(branch-protected)` ([landing.md](./landing.md#what-goes-to-a-person)), and so
does a base requiring a status check that runs only on pull requests, which no pushed candidate can carry. A base that
merely requires status checks the candidate has already passed admits the advance, since the candidate's checks ran on
the very revision that lands.

The `changes(closed, base, head=<branch>)` lookup `merge-on-checks` uses names its head as GitHub requires it,
`<owner>:<branch>`; the adapter qualifies a bare branch with the repository's owner.

## The mock

`blizzard-mock`'s forge (`blizzard-mock/src/blizzard_mock/forge/`) serves what the proofs need, over the git backend it
already has. Its forge today answers `PATCH git/refs` with a real fast-forward-only check (`domain/service.py`,
`update_ref`), so `advance_ref` needs nothing new there; the rest it grows:

- **Branch merges and refs** — `POST /repos/{o}/{r}/merges` (201, 204, and 409 on a conflict), `POST git/refs`, and
  `DELETE git/refs/heads/{b}`. `update_ref`'s docstring stops calling its fast-forward check a compare-and-swap.
- **Pull listings that honour their parameters** — `list_pulls` (`api/pulls.py`, `internal/state_store.py`) today takes
  only `state` and answers every pull oldest first by number. It learns `base`, `head` (as `<owner>:<branch>`), `sort`
  (`created` or `updated`), `direction`, `page`, and `per_page`, and answers GitHub's `Link` header with `next` and
  `last` when a page is not the last. Each pull already carries an `updated_at` that closing and merging move
  (`set_pull_state` and `_record_merge` in `domain/service.py`); ordering and paging by it is what is new, so the
  recently-closed listing is exercised as GitHub answers it. With no `sort` it answers as GitHub does, newest first; the
  hub's adapter always names `sort` and `direction`, so nothing in `blizzard` leans on the default, and any test in
  `blizzard-mock` that relied on oldest first names its order instead.
- **Checks per ref, several and named** — today a check run is derived from the levers of the open pull request whose
  head matches the ref (`list_check_runs`, `domain/service.py`), it is always one run named `ci`, and a revision with no
  pull request reads green at once. A per-ref check lever arms named check runs against any revision or branch, each
  `pending`, `failed`, or `success`, or arms none at all, and a lever set later adds a run beside those already there,
  so a fast-forward candidate on a `-landing` branch can wait, fail its own check, fail only what the base fails, or
  carry no check, and a revision whose one check finished green can gain a second, pending one. `base_checks_failed`
  matches by the resolved revision of the repository's default branch rather than by the ref's spelling, so a check read
  of the base's tip by its sha reads red, as GitHub's would.
- **Every merge method** — `merge_pull` makes a squash commit for `squash` where today it falls through to a merge
  commit, and the `externally_merged` lever takes a method, so a person's squash or rebase merge lands a revision that
  is neither the change's head nor a merge of it, as on GitHub.
- **Protection levers** — one refusing `PATCH git/refs` on a branch with GitHub's 422, and one refusing `POST git/refs`
  and `POST /merges` on branches matching a pattern with GitHub's "Repository rule violations found", so
  `branch-protected` is provable for the base and for a `-landing` branch.
- **A merge-method lever**, under which `merge_pull` refuses the `merge` method with GitHub's 405 "Merge commits are not
  allowed on this repository", so `merge-method-refused` is provable.
- **Conditional reads** — every `GET` answers an `ETag` derived from its response body, so it changes exactly when the
  answer does, and a `304 Not Modified` to a matching `If-None-Match`, uncounted against the rate headers, so the
  cheaper pass is provable.
- **Rate headers on every answer that counts** — GitHub's `X-RateLimit-Limit`, `-Remaining`, `-Reset`, and `-Resource` —
  with the remaining counting down by one on every answer that is not a `304`, per token the request carries, and
  restored to the limit when the window resets, an hour by default. A lever sets the remaining and the reset, and a
  second answers a secondary limit, a 403 whose remaining is not spent and whose body names it, so the budget's
  thresholds and the forges epic's retry rules are provable. The `rate_limited` lever (`api/middleware.py`) answers a
  real reset where it answers `X-RateLimit-Reset: 0` today.

### What the mock must keep

`blizzard`'s upper tiers run against `blizzard-mock`'s `master` as it moves, and `blizzard:rehearsal` runs every
supported release's own hub against it, v0.2.0's among them, which still lands through the `run:` scripts. So from the
moment this lands, the mock serves three clients at once — today's `land`, v0.2.0's `land_common.py`, and the seam this
epic grows — and the first two must go on landing exactly as they do. Three of the additions above touch what they read,
and each is held to that:

- **The page a listing answers.** Today's and v0.2.0's `land_common.py` read the open pull requests into a repository
  with neither `per_page` nor `sort`, and trust the one page they get. A listing that names no `per_page` therefore
  answers every match on one page, as the mock does today, rather than GitHub's thirty; only a request that names
  `per_page` is paged. The hub's adapter always names it, so the seam is still exercised as GitHub pages.
- **What the base's red check matches.** Matching `base_checks_failed` by the default branch's resolved revision, rather
  than its spelling, widens what reads red to any ref at the base's tip. Every scenario that arms the lever in today's
  suites, and in each supported release's, must read as it reads now, and the change is proven against them before it
  merges.
- **The default rate limit.** Counting down never refuses on its own: only `rate_limited` and the secondary-limit lever
  refuse, so a release that reads no rate headers is never stopped by one. A hub that does read them — this epic's
  budget — turns its sweeps away as the remaining falls, so the default limit is set above what the busiest of today's
  suites, `blizzard:journey` and `blizzard:crash-sweep` among them, spends on one token in an hour, measured when the
  mock grows it, and a run that needs a low remaining asks for it with the lever.

The forges epic grows the same mock — mergeable states it cannot yet show, a conflicting `update-branch` answered as
GitHub answers it, a check run that failed to start — in the same `domain/service.py` and `domain/levers.py`. The two
are built one after the other, never side by side, whichever lands first; this epic's mock work may land ahead of the
actions, forges, and flows epics, since nothing in it reads the hub's code.
