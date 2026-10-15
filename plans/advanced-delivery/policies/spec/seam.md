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

`IForgeChecks` gains `change_checks(number)` beside `checks(ref)`, under `checks.read`: the checks that decide whether a
change may merge. On GitHub those are the change's head's; a forge that runs a change's checks on something other than
its head answers from there, so `merge-on-checks` asks the change rather than guessing where its checks ran.
`update_from_base` answers the forges epic's `Update(outcome, method)`, where `method` says how the forge brought the
change up to its base — `merge` on GitHub — so the head gate knows what it is reading
([landing.md](./landing.md#the-three-policies)).

The four new `IForgeRefs` operations and `change_checks` join `OPERATION_CAPABILITY` beside the forges epic's, so
`ForgeRegistry` refuses each where its capability is not offered. A forge kind that cannot move a branch outside a
change offers none of the four, and its repositories cannot fast-forward ([record.md](./record.md#capabilities)).

## The GitHub adapter

The GitHub adapter offers every new capability. `merge_into` is `POST /repos/{o}/{r}/merges`, 201 a merge, 204 nothing
to merge, 409 a conflict; it merges into the landing branch, so the merge commit's first parent is the branch it lands
on. `create_ref` is `POST git/refs` and `delete_ref` `DELETE git/refs/heads/{b}`. `advance_ref` is the GraphQL
`updateRefs` mutation with `beforeOid` set to `expected`, since REST's `PATCH git/refs` only refuses a non-fast-forward
and cannot say what the ref must have been; the adapter reads the repository's node id once per adapter and posts to the
forge's GraphQL endpoint, derived from `api_base` (`https://api.github.com/graphql`, or `<origin>/api/graphql` for a
GitHub Enterprise server). `change_checks` is `checks` of the change's head revision. A base protected so that only pull
requests may change it refuses the advance; the adapter answers `ForgeRejected("protected")`, which `fast-forward`
answers as `NeedsPerson(base-protected)` ([landing.md](./landing.md#what-goes-to-a-person)).

## The mock

`blizzard-mock`'s forge serves what the proofs need, over the git backend it already has:

- **GitHub additions** — `POST /repos/{o}/{r}/merges`, `POST` and `DELETE` on `git/refs`, and a `/graphql` endpoint
  answering `updateRefs` with a real compare-and-swap. `PATCH git/refs`'s docstring stops calling its fast-forward check
  compare-and-swap.
- **A protected-branch lever**, so `base-protected` is provable.
- **Rate headers on every answer** — GitHub's `X-RateLimit-*` with `X-RateLimit-Resource` — with a lever setting the
  remaining and the reset, and a second answering a secondary limit, a 403 whose remaining is not spent, so the budget's
  thresholds and the forges epic's retry rules are provable.
- **Its `update-branch` conflict** answered 422 with a message naming the conflict, as GitHub answers it, where it
  answers 405 today, so the land action's conflict path is exercised against the mock and not only on a later poll.
