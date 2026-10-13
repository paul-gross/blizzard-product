# GitLab, and the seam it shares

## The seam grows

The forges slice split the seam by the caller that needs each part, and this slice grows each role only where a policy
needs it, every new operation behind its own capability. `IForgeRefs` (`hub/forge/landing.py`) gains the four
`fast-forward` needs:

| Operation                                 | Capability       | Answers                                                                                                                               |
| ----------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `advance_ref(branch, expected, revision)` | `refs.advance`   | Moves `branch` from `expected` to `revision`, a fast-forward; `ForgeRejected("stale")` when it was not at `expected`                  |
| `merge_into(branch, revision, message)`   | `branches.merge` | The merge commit on `branch` bringing `revision` in, `None` when `branch` already holds it, `ForgeRejected("conflict")` on a conflict |
| `create_ref(branch, revision)`            | `refs.create`    | Creates `branch` at `revision`; an existing branch already at `revision` is not an error                                              |
| `delete_ref(branch)`                      | `refs.delete`    | Deletes `branch`; a branch already gone is not an error                                                                               |

`IForgeChecks` gains `change_checks(number)` beside `checks(ref)`, under `checks.read`: the checks that decide whether a
change may merge, which on some forges run on something other than the change's head
([The GitLab adapter](#the-gitlab-adapter)). `update_from_base` answers the forges slice's `Update(outcome, method)`,
where `method` is `merge` or `rebase`, since the two kinds of forge bring a change up to its base differently
([The head gate](#the-head-gate)).

The GitHub adapter offers every new capability. `merge_into` is `POST /repos/{o}/{r}/merges`, 201 a merge, 204 nothing
to merge, 409 a conflict; it merges into the landing branch, so the merge commit's first parent is the branch it lands
on. `create_ref` is `POST git/refs` and `delete_ref` `DELETE git/refs/heads/{b}`. `advance_ref` is the GraphQL
`updateRefs` mutation with `beforeOid` set to `expected`, since REST's `PATCH git/refs` only refuses a non-fast-forward
and cannot say what the ref must have been; the adapter reads the repository's node id once per adapter and posts to the
forge's GraphQL endpoint, derived from `api_base` (`https://api.github.com/graphql`, or `<origin>/api/graphql` for a
GitHub Enterprise server). `change_checks` is `checks` of the change's head revision. The four new operations join
`OPERATION_CAPABILITY` beside the forges slice's, so `ForgeRegistry` refuses each where its capability is not offered. A
base protected so that only pull requests may change it refuses the advance; the adapter answers
`ForgeRejected("protected")`, which `fast-forward` answers as `NeedsPerson(base-protected)`
([landing.md](./landing.md#what-goes-to-a-person)).

## The GitLab adapter

`KNOWN_FORGE_KINDS` becomes `{"github", "gitlab"}`. A work source still cannot name a GitLab forge — its providers stay
`{"github"}`, and `forge_kind_mismatch` refuses it — since GitLab as a source of work is not this epic's. The adapter
supports GitLab 15.6 and later, the first to answer `detailed_merge_status`, and a forge whose server reports an older
version on its first call is refused `forge_version_unsupported` naming the version it found.

`GitLabLanding` (`hub/forge/internal/gitlab_landing.py`) speaks GitLab's REST API under the forge's `api_base`
(`https://gitlab.com/api/v4`), asynchronously as every adapter does, authenticated with `PRIVATE-TOKEN: <secret>`, the
project addressed by its URL-encoded full path and every branch name URL-encoded, since both may hold `/`. It pages with
`per_page=100` to the same bounds the GitHub adapter keeps.

| Operation          | GitLab                                                                                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `branch_tip`       | `GET /projects/:id/repository/branches/:branch`                                                                                          |
| `compare`          | `GET /projects/:id/repository/compare?from=&to=&straight=false`, and the reverse for `behind`; `merge_base` from `repository/merge_base` |
| `commit_exists`    | `GET /projects/:id/repository/commits/:sha`                                                                                              |
| `changes`          | `GET /projects/:id/merge_requests?state=opened\|merged\|closed&target_branch=&source_branch=`                                            |
| `open_change`      | `POST /projects/:id/merge_requests`                                                                                                      |
| `change`           | `GET /projects/:id/merge_requests/:iid`                                                                                                  |
| `update_from_base` | `PUT /projects/:id/merge_requests/:iid/rebase`, answered `rebase`; progress read back with `include_rebase_in_progress=true`             |
| `merge`            | `PUT /projects/:id/merge_requests/:iid/merge` with `sha` as the expected head, which GitLab checks; `squash` for that method             |
| `checks`           | `GET /projects/:id/repository/commits/:sha/statuses` — the latest status of each of the revision's jobs and external statuses            |
| `change_checks`    | The change's `head_pipeline`, read from `change`, and its jobs from `GET /projects/:id/pipelines/:pipeline_id/jobs`                      |
| `rerun_check`      | `POST /projects/:id/jobs/:job_id/retry`; an external status cannot be re-run and answers `False`                                         |

`checks` reads statuses without `all=true`, so a job that failed and was retried counts once, at its latest outcome, as
a GitHub check run does. `change_checks` reads the merge request's own pipeline rather than the head's, because a
project running merged-results pipelines, or one whose pipelines run only for merge requests, runs its checks on a merge
ref GitLab builds, not on the head; reading the head would find nothing there and wait forever. Retried jobs are left
out of a pipeline's jobs by default, so the same latest-outcome rule holds.

It offers `refs.read`, `compare`, `commits.read`, `changes.open`, `changes.read`, `changes.update-from-base`,
`changes.merge.merge`, `changes.merge.squash`, `checks.read`, and `checks.rerun`. It offers none of `refs.advance`,
`branches.merge`, `refs.create`, or `refs.delete` for landing, because GitLab's API has no way to move a branch to a
revision or to merge one branch into another outside a merge request. A GitLab project's merge method — a merge commit,
semi-linear, or fast-forward — is the project's setting rather than a request's, so `changes.merge.merge` there means
"merge by the project's own method", and the change's `merged_revision` says what landed; `changes.merge.rebase` is not
offered.

Its vocabulary maps onto the seam's and nothing else crosses:

- **`state`** — `opened` → `open`, `merged` → `merged`, `closed` → `closed`. `locked` is the brief state of a merge
  request GitLab is in the middle of merging, so it maps to `open` with mergeability `pending`, and the next read sees
  it merged.
- **`merged_revision`** — `merge_commit_sha`, else `squash_commit_sha`, else, for a project merging fast-forward, the
  change's head `sha`.
- **`mergeability`**, from `detailed_merge_status` — `mergeable` → `mergeable`; `need_rebase` → `behind`; `conflict` and
  `broken_status` → `conflicting`; `ci_must_pass`, `ci_still_running`, `not_approved`, `draft_status`,
  `discussions_not_resolved`, `merge_request_blocked`, `status_checks_must_pass`, and anything else it declines on →
  `blocked`; `checking`, `unchecked`, `preparing`, and `approvals_syncing` → `pending`.
- **A check's `outcome`** — `success` → `passed`; `failed` → `failed`, or `neutral` when the job allows failure;
  `skipped` and `manual` → `neutral`; `canceled` → `pending`, as GitHub's `cancelled` is read, so a cancelled pipeline
  is waited out rather than charged; every running or queued status → `pending`. `detail` carries GitLab's own word and
  `details_url` the job's `web_url`.
- **Failures** follow the forges slice's retry rules: 404 → `ForgeNotFound`; 401, and a 403 that is not a rate limit, →
  `ForgeRejected("auth")`; 429 → `ForgeRateLimited`; 405, 406, 409, and 422 on a merge are re-read and, with the change
  still open, answer `pending`; 5xx and timeouts → `ForgeUnavailable`.

`ForgeRegistry` picks the adapter by the forge's `kind`, and the capability check, the policies, and the combining step
never learn which they hold.

## The head gate

GitHub brings a change up to its base by merging the base in; GitLab rebases it, rewriting the head, so the head no
longer descends from the submitted commit and today's gate would call it foreign. GitLab's comparisons cannot prove a
rebase carried the work unchanged the way `base_merge.py` proves a base merge, because they carry no blob ids, no
per-file line counts, and no total that would show a commit list cut short. So a rebased head is admitted only on the
hub's own word, checked against what GitLab can say:

1. When `update_from_base` answers `rebase`, `land` hands `rebase-requested/<repo>/<head>` to its effects, naming the
   head it rebased from, before it asks the forge to act.
2. `gate_head` admits a later head that does not descend from the submitted commit only when such a marker names the
   head it replaced, the change's `diff_refs.base_sha` is the base's current tip, the rebased head carries as many
   commits past that base as the replaced head carried past its own, and the set of paths it changes is the set the
   replaced head changed.

A head that fails any of these — a rebase nobody asked the hub for, a person's push in between, a rebase that dropped or
added a file — is a foreign head, and `failure`, as today. The check trades the proof GitHub's base merges get for one
the forge can answer, and it stands on the narrow fact that the hub asked for this rebase and GitLab performed it.

## Where a GitLab repository lives

GitLab groups nest, so a GitLab project's owner is a path (`group/subgroup`). A GitLab repository record's `owner` holds
the full namespace, and `validate_fields` admits `/` in `owner` when the record's forge is GitLab. `RepoRef`
(`foundation/repo_ref.py`, its cases in `tests/test_repo_ref.py`) parses a nested origin's owner as its immediate parent
today; it gains `namespace`, every segment between host and name, and `CommitOrigin` carries it, so `_candidates`
matches a GitLab record's owner against the origin's whole namespace and a GitHub record's against its one segment, as
now. `repo_identity` and `pointer_key` in the same module key by the whole namespace too, so `a/x/r` and `b/x/r` on one
host stay two repositories. Both run on the runner as well as the hub, so the runner and hub ship the change together,
and a GitHub origin, whose namespace is its one owner, keys exactly as it does today.

`hub/forge/web.py` learns GitLab's web shapes — `/-/tree/<branch>`, `/-/commit/<sha>`, `/-/merge_requests/<iid>`, and a
web origin derived from `api_base` by dropping `/api/v4` — and `delivery_read.py`'s `_commit_url`, which today parses a
GitHub pull request's address, asks `web.py` for a commit's address from the repository's forge instead.

## The mock

`blizzard-mock`'s forge serves what the proofs need, over the git backend it already has:

- **GitHub additions** — `POST /repos/{o}/{r}/merges`, `POST` and `DELETE` on `git/refs`, and a `/graphql` endpoint
  answering `updateRefs` with a real compare-and-swap. `PATCH git/refs`'s docstring stops calling its fast-forward check
  compare-and-swap.
- **A GitLab face** — `/api/v4` routes for every operation above, over the same repositories, with its own levers for
  `need_rebase`, a rebase that conflicts, a failing pipeline, a merged-results pipeline that runs on a merge ref, a
  retried job, and a merge request passing through `locked`.
- **A protected-branch lever**, so `base-protected` is provable.
- **Rate headers on every answer** — GitHub's `X-RateLimit-*` with `X-RateLimit-Resource`, GitLab's `RateLimit-*` — with
  a lever setting the remaining and the reset, and a second answering a secondary limit, a 403 whose remaining is not
  spent, so the budget's thresholds and the forges slice's retry rules are provable.
- **Its `update-branch` conflict** answered 422 with a message naming the conflict, as GitHub answers it, where it
  answers 405 today, so the land action's conflict path is exercised against the mock and not only on a later poll.

The mock is written from GitLab's documentation, and the behaviours above are the ones it would get wrong silently. A
contract suite therefore runs the GitLab adapter against a real GitLab CE container, pinned at the supported minimum,
covering every operation, the `locked` state, a retried job, a merged-results pipeline, and a rebase. Booting GitLab
takes minutes and needs docker, so the suite is a local-only `mise run gitlab-contract`, as the image smoke is
(`blizzard-context:/verification/blizzard/commands/packaging.md`), declared as its own method in blizzard's
verifiability matrix and owed by any change to the GitLab adapter.
