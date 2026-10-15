# The GitLab adapter

## A second kind

`KNOWN_FORGE_KINDS` becomes `{"github", "gitlab"}`, so a forge record may name `gitlab`. A work source still cannot name
a GitLab forge — its providers stay `{"github"}`, and `forge_kind_mismatch` refuses it — since GitLab as a source of
work is not this initiative's. `ForgeRegistry` builds the adapter by the forge's `kind`, through the factory the
composition root supplies, and the capability check, the policies, and the combining step never learn which kind they
hold.

The adapter supports GitLab 15.6 and later, the first to answer `detailed_merge_status`. It reads the server's version
(`GET /version`) on its first call, and a server reporting an older one is refused `forge_version_unsupported`, naming
the version it found, before any landing call is made; the refusal escalates the visit to a person with cause
`forge-config`, as the forges epic escalates a forge the hub cannot use.

## Its operations

`GitLabLanding` (`hub/forge/internal/gitlab_landing.py`) speaks GitLab's REST API under the forge's `api_base`
(`https://gitlab.com/api/v4`), asynchronously as every adapter does, authenticated with `PRIVATE-TOKEN: <secret>`, the
project addressed by its URL-encoded full path ([namespaces.md](./namespaces.md)) and every branch name URL-encoded,
since both may hold `/`. It pages with `per_page=100` to the same bounds the GitHub adapter keeps.

| Operation          | GitLab                                                                                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `branch_tip`       | `GET /projects/:id/repository/branches/:branch`                                                                                          |
| `compare`          | `GET /projects/:id/repository/compare?from=&to=&straight=false`, and the reverse for `behind`; `merge_base` from `repository/merge_base` |
| `commit_exists`    | `GET /projects/:id/repository/commits/:sha`                                                                                              |
| `changes`          | `GET /projects/:id/merge_requests?state=opened\|merged\|closed&target_branch=&source_branch=`                                            |
| `open_change`      | `POST /projects/:id/merge_requests`                                                                                                      |
| `change`           | `GET /projects/:id/merge_requests/:iid`                                                                                                  |
| `update_from_base` | `PUT /projects/:id/merge_requests/:iid/rebase`, answered with method `rebase`; progress read back with `include_rebase_in_progress=true` |
| `merge`            | `PUT /projects/:id/merge_requests/:iid/merge` with `sha` as the expected head, which GitLab checks; `squash` for that method             |
| `checks`           | `GET /projects/:id/repository/commits/:sha/statuses` — the latest status of each of the revision's jobs and external statuses            |
| `change_checks`    | The change's `head_pipeline`, read from `change`, and its jobs from `GET /projects/:id/pipelines/:pipeline_id/jobs`                      |
| `rerun_check`      | `POST /projects/:id/jobs/:job_id/retry`; an external status cannot be re-run and answers `False`                                         |

`park`'s recently-closed listing (`ForgeRegistry.recently_closed`) reads `GET /projects/:id/merge_requests` into the
base with `state=all&order_by=updated_at&sort=desc`, since GitLab keeps merged and closed apart as two states, and keeps
the merged and closed among them; a change parked for a week and then merged is at the top of the listing however busy
the project.

`checks` reads statuses without `all=true`, so a job that failed and was retried counts once, at its latest outcome, as
a GitHub check run does. `change_checks` reads the merge request's own pipeline rather than the head's, because a
project running merged-results pipelines, or one whose pipelines run only for merge requests, runs its checks on a merge
ref GitLab builds, not on the head; reading the head would find nothing there and wait forever. Retried jobs are left
out of a pipeline's jobs by default, so the same latest-outcome rule holds.

## What it offers

It offers `refs.read`, `compare`, `commits.read`, `changes.open`, `changes.read`, `changes.update-from-base`,
`changes.merge.merge`, `changes.merge.squash`, `checks.read`, and `checks.rerun`. It offers none of `refs.advance`,
`branches.merge`, `refs.create`, or `refs.delete` for landing, because GitLab's API has no way to move a branch to a
revision or to merge one branch into another outside a merge request. A GitLab project's merge method — a merge commit,
semi-linear, or fast-forward — is the project's setting rather than a request's, so `changes.merge.merge` there means
"merge by the project's own method", and the change's `merged_revision` says what landed; `changes.merge.rebase` is not
offered.

## Its vocabulary

Its words map onto the seam's and nothing else crosses:

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

## Its failures

Failures follow the forges epic's retry rules ([seam.md](../../forges/spec/seam.md)): 404 → `ForgeNotFound`; 401, and a
403 that is not a rate limit, → `ForgeRejected("auth")`; 429 → `ForgeRateLimited`, its `resume_at` from
`RateLimit-Reset` or `Retry-After`; 405, 406, 409, and 422 on a merge are re-read and, with the change still open,
answer `pending`; a rebase GitLab answers with a conflict → `Update(outcome=conflict)`; 5xx and timeouts →
`ForgeUnavailable`.

## Its rate budget

The policies epic's `ForgeBudgets` keys each budget by `(forge_id, secret_name, resource)`; a GitLab forge has one
bucket, recorded as `core`. The adapter's client installs the same `httpx` event hook the GitHub adapter does, reading
GitLab's `RateLimit-Limit`, `-Remaining`, and `-Reset`. A self-managed server that sends those headers only on the
response that throttles teaches the budget only from its refusals, and its per-minute window makes the wait that follows
a short one. gitlab.com allows about two thousand authenticated requests a minute, and the policies epic's cheap pass
keeps a GitLab landing well inside it without conditional requests. Because GitLab's windows are a minute long,
`forge-budget-low` and `forge-rate-limited` are each recorded at most once an hour for a GitLab budget, rather than once
per window, so a busy hour does not fill the event log.
