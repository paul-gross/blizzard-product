# The seam the policies grow

## The seam grows

The forges epic split the seam by the caller that needs each part, and this epic grows each role only where a policy
needs it, every new operation behind its own capability. `IForgeRefs` (`hub/forge/landing.py`) gains the four
`fast-forward` needs:

| Operation                                 | Capability       | Answers                                                                                                                               |
| ----------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `advance_ref(branch, expected, revision)` | `refs.advance`   | Moves `branch` from `expected` to `revision`, a fast-forward; `ForgeRejected("stale")` when it was not at `expected`                  |
| `merge_into(branch, revision, message)`   | `branches.merge` | The merge commit on `branch` bringing `revision` in, `None` when `branch` already holds it, `ForgeRejected("conflict")` on a conflict |
| `create_ref(branch, revision)`            | `refs.create`    | Creates `branch` at `revision`; an existing branch already at `revision` is not an error                                              |
| `delete_ref(branch)`                      | `refs.delete`    | Deletes `branch`; a branch already gone is not an error                                                                               |

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
  does not descend from refuses the advance. The adapter answers a refusal by its message: GitHub's 422 "Update is not a
  fast forward" is `ForgeRejected("stale")`; a 422 naming a protected branch ("Protected branch update failed", "Cannot
  update this protected ref", or a required status check the update lacks) is `ForgeRejected("protected")`. `expected`
  is what the adapter compares the branch's tip against before it sends the update, so a tip it already knows has moved
  is answered `stale` without a write.
- **`merge_into`** is `POST /repos/{o}/{r}/merges`, 201 a merge, 204 nothing to merge, 409 a conflict; it merges into
  the landing branch, so the merge commit's first parent is the branch it lands on.
- **`create_ref`** is `POST git/refs` and **`delete_ref`** `DELETE git/refs/heads/{b}`; a 422 "Reference already exists"
  at the same revision and a 422 or 404 on a branch already gone are not errors.
- **`recently_closed`** is `GET pulls?state=closed&base=<base>&sort=updated&direction=desc&per_page=<limit>`, one page.
  The adapter keeps the answer for `PARK_POLL_FLOOR` ([landing.md](./landing.md#awaiting-a-merge)). Adapters are built
  and cached per repository ([the forges epic's registry](../../forges/spec/seam.md#the-registry)), so every parked
  change on a repository is answered from one listing, and the cache lives inside the adapter, never in an action.
- **`change_checks`** is `checks` of the change's head revision.

A base protected so that only pull requests may change it refuses the advance as `ForgeRejected("protected")`, which
`fast-forward` answers as `NeedsPerson(base-protected)` ([landing.md](./landing.md#what-goes-to-a-person)). A base that
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
  only `state` and answers every pull oldest first. It learns `base`, `head` (as `<owner>:<branch>`), `sort` (`created`
  or `updated`), `direction`, `page`, and `per_page`, and keeps an `updated_at` per pull that every state change moves,
  so the recently-closed listing and its paging are exercised as GitHub answers them.
- **Checks per ref** — today a check run is derived from the levers of the open pull request whose head matches the ref
  (`list_check_runs`), and a revision with no pull request reads green at once. A per-ref check lever arms `pending`,
  `failed`, or no check at all against any revision or branch, so a fast-forward candidate on a `-landing` branch can
  wait, fail its own check, fail only what the base fails, or carry no check.
- **Every merge method** — `merge_pull` makes a squash commit for `squash` where today it falls through to a merge
  commit, and the `externally_merged` lever takes a method, so a person's squash or rebase merge lands a revision that
  is neither the change's head nor a merge of it, as on GitHub.
- **A protected-branch lever**, which refuses `PATCH git/refs` on the branch with GitHub's 422, so `base-protected` is
  provable.
- **Conditional reads** — every `GET` answers an `ETag` and a `304 Not Modified` to a matching `If-None-Match`,
  uncounted against the rate headers, so the cheaper pass is provable.
- **Rate headers on every answer** — GitHub's `X-RateLimit-Limit`, `-Remaining`, `-Reset`, and `-Resource` — with a
  lever setting the remaining and the reset, and a second answering a secondary limit, a 403 whose remaining is not
  spent and whose body names it, so the budget's thresholds and the forges epic's retry rules are provable. The
  `rate_limited` lever (`api/middleware.py`) answers a real reset where it answers `X-RateLimit-Reset: 0` today.
