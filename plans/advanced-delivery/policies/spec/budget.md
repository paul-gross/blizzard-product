# One rate budget per key

## Why now

Nothing in the hub reads a forge's rate limit today, and the call that spends it fastest is landing itself. A
merge-on-checks pass costs about fourteen calls per repository: the closed pull requests scanned up to ten pages deep to
learn whether the branch already merged, the open ones listed, the pull request read, the head gate's compares, and the
head's and base's check runs. The shipped graphs poll every thirty seconds, so a single delivering repository spends
about 1,700 calls an hour, and three landing at once exhaust a GitHub key's 5,000. Parked changes polled for days,
GitLab's tighter limits, and the forge-status sweep and close drain sharing a credential with landing all add to that.
This is the slice where the hub must make a pass cheap, and then slow itself down before the forge stops it.

## The cost of a pass

Most of a pass's calls ask a question whose answer has not changed since the last pass, so the hub stops paying for
them:

- **The one costly lookup becomes one call.** Whether the chunk's branch already merged is
  `changes(closed, base, head=<branch>)`, a filtered read, in place of paging through every closed pull request into the
  base.
- **Unchanged answers are free.** The GitHub adapter keeps each `GET`'s `ETag` per URL, in memory per adapter, and sends
  it back as `If-None-Match`; a `304 Not Modified` answers from what it kept, and GitHub does not count a 304 against
  the key's limit.
- **What cannot change is read once.** A compare between two fixed revisions, and the checks of a revision whose every
  check has completed, are kept per adapter until the hub itself asks for a re-run, so the head gate and the verdict
  read them once per revision rather than once per pass.
- **A pass with nothing new costs almost nothing.** With the reads above, a pass in which neither the head nor the base
  moved and the checks are still running spends its calls on 304s and the one check read whose answer moved.
- **The shipped graphs poll every minute.** Their delivery nodes' `poll_interval` goes from 30 to 60 seconds; a CI run
  takes minutes, and a minute's cadence lands within a minute of green.

The arithmetic, for one key on GitHub: a delivering repository in its steady state costs at most one counted call a
pass, sixty an hour, and a pass where something moved — a new head, a moved base, a finished check — costs four or five.
A chunk's handful of writes (opening the change, an update, a re-run, the merge) count once each. A repository with
parked changes costs its one recently-closed listing every two minutes, thirty an hour, mostly answered 304, however
many changes are parked on it. So a key comfortably carries several dozen repositories delivering at once, where today
it carries three. GitLab counts by the minute — gitlab.com allows about two thousand authenticated requests a minute —
and the same reads keep a pass well inside it without conditional requests.

## The budget

`ForgeBudgets` (`hub/forge/budget.py`) is held per tenant beside `ForgeRegistry` and keyed by forge, credential, and the
forge's own rate bucket — `(forge_id, secret_name, resource)` — where `resource` is GitHub's `X-RateLimit-Resource`,
`core` or `graphql`, and `core` on GitLab. Keying by bucket keeps `advance_ref`'s GraphQL call from overwriting what the
hub knows of the REST budget. A forge counts what each user spends, and two secrets holding the same user's tokens share
a limit the hub sees twice; the budget reads the forge's own headers rather than counting, so each reading is still the
truth about that limit, and the hub only learns of a shared spend one answer late.

Every caller that holds a key on a forge shares that key's budget: the landing seam's adapters, the parked-change
listing, and the work-source family (`IWorkSource`, `IWorkAnnotator`, `IWorkCloser`), whose adapters
`StoreWorkSourceRegistry` now builds with their forge's budget.

A budget learns from the forge, never from a count of its own. Every response's rate headers update it — GitHub's
`X-RateLimit-Limit`, `-Remaining`, `-Reset`, and `-Resource`; GitLab's `RateLimit-Limit`, `-Remaining`, and `-Reset` —
through an `httpx` event hook every adapter's client installs. What counts as the forge refusing for rate is the forges
slice's retry rules, which already raise `ForgeRateLimited(resume_at)` for a spent remaining, a `Retry-After`, or a
secondary limit; the budget hears of each one and marks itself spent until `resume_at`. A GitLab server that reports its
rate headers only when it throttles teaches the budget only from its refusals, and its per-minute window makes the wait
that follows a short one. A budget the hub has not yet heard from admits every caller; it is held in memory, and a
restarted hub relearns it from its first answer.

## Who goes first

Each caller says how much the call matters, and a budget running low turns the least important away first, before any
request is made:

| Priority   | Callers                                                                       | Admitted while the remaining budget is |
| ---------- | ----------------------------------------------------------------------------- | -------------------------------------- |
| `landing`  | A `land` pass for a policy the hub lands itself                               | above nothing — until spent            |
| `watching` | A parked change's poll, the recently-closed listing                           | above a fifth of the limit             |
| `sweeping` | The forge-status annotation sweep, the close drain, the garden's commit check | above two fifths of the limit          |

A call turned away raises `ForgeBudgetLow(resume_at)` without reaching the forge, and each caller slows rather than
fails: a `land` pass answers `pending` and a parked poll `awaiting-merge`, each with `Pass.next_poll_at` set to
`resume_at`, so its next poll is no earlier; the annotation sweep skips its pass; the close drain defers the intent to
`resume_at` without spending its failure backoff (`hub/domain/work_items/closure.py`); the garden's commit check answers
that the forge could not say, as it does for an unavailable one. A `land` pass that waits on the budget waits inside its
node's poll timeout like any `pending`, and the three-hour timeout is what keeps a reset an hour away from bouncing it.

## What an operator sees

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `forge-budget-low` (warning), recorded once per
budget per reset window the first time a caller is turned away, and `forge-rate-limited` (warning), recorded once per
window when the forge itself refuses. GitLab's windows are a minute long, so each is recorded at most once an hour for a
GitLab budget. Each names the forge, the secret, the bucket, the remaining and limit it last read, and the reset.
`tests/event_log_kind_census.py` and `blizzard-context:/domain/operations.md` gain both. A forge's document on
`GET /api/forges/{forge}` carries each of its budgets' last reading, so the admin screen's forge detail shows how close
each key runs to its limit.
