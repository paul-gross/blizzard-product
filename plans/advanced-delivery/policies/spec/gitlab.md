# GitLab, and the seam it shares

## The seam grows

`IForgeLanding` (`hub/forge/landing.py`) gains the two operations `fast-forward` needs, each behind its own capability:

| Operation                                 | Capability       | Answers                                                                                                                               |
| ----------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `advance_ref(branch, expected, revision)` | `refs.advance`   | Moves `branch` from `expected` to `revision`, a fast-forward; `ForgeRejected("stale")` when it was not at `expected`                  |
| `merge_into(branch, revision, message)`   | `branches.merge` | The merge commit on `branch` bringing `revision` in, `None` when `branch` already holds it, `ForgeRejected("conflict")` on a conflict |

`update_from_base` answers `Update(accepted, method)`, where `method` is `merge` or `rebase`, since the two kinds of
forge bring a change up to its base differently ([The head gate](#the-head-gate)).

The GitHub adapter offers both new capabilities. `merge_into` is `POST /repos/{o}/{r}/merges`, 201 a merge, 204 nothing
to merge, 409 a conflict. `advance_ref` is the GraphQL `updateRefs` mutation with `beforeOid` set to `expected`, since
REST's `PATCH git/refs` only refuses a non-fast-forward and cannot say what the ref must have been; the adapter reads
the repository's node id once per adapter and posts to the forge's GraphQL endpoint, derived from `api_base`
(`https://api.github.com/graphql`, or `<origin>/api/graphql` for a GitHub Enterprise server). A base protected so that
only pull requests may change it refuses the advance; the adapter answers `ForgeRejected("protected")`, which `land`
routes as `failure` with findings naming the protection, since nothing mechanical gets past it.

## The GitLab adapter

`KNOWN_FORGE_KINDS` becomes `{"github", "gitlab"}`. A work source still cannot name a GitLab forge — its providers stay
`{"github"}`, and `forge_kind_mismatch` refuses it — since GitLab as a source of work is not this epic's.

`GitLabLanding` (`hub/forge/internal/gitlab_landing.py`) speaks GitLab's REST API under the forge's `api_base`
(`https://gitlab.com/api/v4`), authenticated with `PRIVATE-TOKEN: <secret>`, the project addressed by its URL-encoded
full path. It pages with `per_page=100` to the same bounds the GitHub adapter keeps.

| Operation          | GitLab                                                                                                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `branch_tip`       | `GET /projects/:id/repository/branches/:branch`                                                                                 |
| `compare`          | `GET /projects/:id/repository/compare?from=&to=&straight=false`, and the reverse for `behind`; `diffs` mapped onto `FileChange` |
| `commit_exists`    | `GET /projects/:id/repository/commits/:sha`                                                                                     |
| `changes`          | `GET /projects/:id/merge_requests?state=opened\|merged\|closed&target_branch=&source_branch=`                                   |
| `open_change`      | `POST /projects/:id/merge_requests`                                                                                             |
| `change`           | `GET /projects/:id/merge_requests/:iid`                                                                                         |
| `update_from_base` | `PUT /projects/:id/merge_requests/:iid/rebase`, answered `rebase`; progress read back with `include_rebase_in_progress=true`    |
| `merge`            | `PUT /projects/:id/merge_requests/:iid/merge` with `sha` as the expected head, which GitLab checks; `squash` for that method    |
| `checks`           | `GET /projects/:id/repository/commits/:sha/statuses?all=true` — the revision's pipeline jobs and external statuses              |
| `rerun_check`      | `POST /projects/:id/jobs/:job_id/retry`; an external status cannot be re-run and answers `False`                                |

It offers `refs.read`, `compare`, `commits.read`, `changes.open`, `changes.read`, `changes.update-from-base`,
`changes.merge.merge`, `changes.merge.squash`, `checks.read`, and `checks.rerun`. It offers neither `refs.advance` nor
`branches.merge`, because GitLab's API has no way to move a branch or to merge one branch into another outside a merge
request, and not `changes.merge.rebase`, whose merge method is a project's setting rather than a request's.

Its vocabulary maps onto the seam's and nothing else crosses:

- **`state`** — `opened` → `open`, `merged` → `merged`, `closed` and `locked` → `closed`.
- **`mergeability`**, from `detailed_merge_status` — `mergeable` → `mergeable`; `need_rebase` → `behind`; `conflict` →
  `conflicting`; `ci_must_pass`, `ci_still_running`, `not_approved`, `draft_status`, `discussions_not_resolved`,
  `merge_request_blocked`, `status_checks_must_pass`, and anything else it declines on → `blocked`; `checking`,
  `unchecked`, and `preparing` → `pending`.
- **A check's `outcome`** — `success` → `passed`; `failed` → `failed`, or `neutral` when the job allows failure;
  `skipped` and `manual` → `neutral`; `canceled` → `pending`, as GitHub's `cancelled` is read today, so a cancelled
  pipeline is waited out rather than charged; every running or queued status → `pending`.
- **Failures** — 404 → `ForgeNotFound`; 405, 409, and 422 → `ForgeRejected` with GitLab's message; 5xx and timeouts →
  `ForgeUnavailable`.

`ForgeRegistry` picks the adapter by the forge's `kind`, and the capability check, the policies, and the combining step
never learn which they hold.

## The head gate

GitHub brings a change up to its base by merging the base in; GitLab rebases it, rewriting the head, so the head no
longer descends from the submitted commit and today's gate would call it foreign. When `update_from_base` answers
`rebase`, `land` writes `rebase-requested/<repo>/<head>` before the forge acts, and `gate_head` admits a later head that
does not descend from the submitted commit only when that marker names the head it was rebased from and the new head's
change against its base carries exactly the submitted change: `base_merge.rebase_carries_only` proves it from compare
payloads, as `contributes_only_base_change` proves a base merge, translating each file's change blocks across the base's
movement and refusing a rename, a truncated file list, or any block it cannot place. A rebase that changed the work is a
foreign head, and `failure`, as today.

## Where a GitLab repository lives

GitLab groups nest, so a GitLab project's owner is a path (`group/subgroup`). A GitLab repository record's `owner` holds
the full namespace, and `validate_fields` admits `/` in `owner` when the record's forge is GitLab. `RepoRef`
(`foundation/repo_ref.py`, its cases in `tests/test_repo_ref.py`) parses a nested origin's owner as its immediate parent
today; it gains `namespace`, every segment between host and name, and `CommitOrigin` carries it, so `_candidates`
matches a GitLab record's owner against the origin's whole namespace and a GitHub record's against its one segment, as
now.

`hub/forge/web.py` learns GitLab's web shapes — `/-/tree/<branch>`, `/-/commit/<sha>`, `/-/merge_requests/<iid>`, and a
web origin derived from `api_base` by dropping `/api/v4` — and `delivery_read.py`'s `_commit_url`, which today parses a
GitHub pull request's address, asks `web.py` for a commit's address from the repository's forge instead.

## The mock

`blizzard-mock`'s forge serves what the proofs need, over the git backend it already has:

- **GitHub additions** — `POST /repos/{o}/{r}/merges`, and a `/graphql` endpoint answering `updateRefs` with a real
  compare-and-swap. `PATCH git/refs`'s docstring stops calling its fast-forward check compare-and-swap.
- **A GitLab face** — `/api/v4` routes for every operation above, over the same repositories, with its own levers for
  `need_rebase`, a rebase that conflicts, and a failing pipeline.
- **A protected-branch lever**, so `ForgeRejected("protected")` is provable.
- **Its `update-branch` conflict** answered 422 with a message naming the conflict, as GitHub answers it, where it
  answers 405 today, so the land action's conflict path is exercised against the mock and not only on a later poll.
