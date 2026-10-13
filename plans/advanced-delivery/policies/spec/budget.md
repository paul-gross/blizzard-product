# One rate budget per forge

## Why now

Nothing in the hub reads a forge's rate limit today. A 403 or 429 from GitHub looks to the land script like a degraded
read and to the work-source adapters like any failed request, and the close drain's only backoff is its own per-intent
retry. That held while landings were few and short. Parked changes polled for days, GitLab's tighter limits, and the
forge-status sweep and close drain sharing a credential with landing make it the slice where the hub must slow itself
down before the forge stops it.

## The budget

`ForgeBudgets` (`hub/forge/budget.py`) is held per tenant beside `ForgeRegistry` and keyed by forge and credential —
`(forge_id, secret_name)` — since a forge counts what each key spends, not what each server serves. Every caller that
holds a key on a forge shares that key's budget: the landing seam's adapters, the parked-change listing, and the
work-source family (`IWorkSource`, `IWorkAnnotator`, `IWorkCloser`), whose adapters `StoreWorkSourceRegistry` now builds
with their forge's budget.

A budget learns from the forge, never from a count of its own. Every response's rate headers update it — GitHub's
`X-RateLimit-Limit`, `-Remaining`, and `-Reset`; GitLab's `RateLimit-Limit`, `-Remaining`, and `-Reset` — through an
`httpx` event hook every adapter's client installs. A 403 or 429 carrying `Retry-After` or a spent remaining marks the
budget spent until its reset and raises `ForgeRateLimited(resume_at)`, a `ForgeUnavailable`, so every caller that
already waits out an unavailable forge waits this out too. A budget the hub has not yet heard from admits every caller;
it is held in memory, and a restarted hub relearns it from its first answer.

## Who goes first

Each caller says how much the call matters, and a budget running low turns the least important away first, before any
request is made:

| Priority   | Callers                                                                       | Admitted while the remaining budget is |
| ---------- | ----------------------------------------------------------------------------- | -------------------------------------- |
| `landing`  | A `land` pass for a policy the hub lands itself                               | above nothing — until spent            |
| `watching` | A parked change's poll, the recently-closed listing                           | above a fifth of the limit             |
| `sweeping` | The forge-status annotation sweep, the close drain, the garden's commit check | above two fifths of the limit          |

A call turned away raises `ForgeBudgetLow(resume_at)` without reaching the forge, and each caller slows rather than
fails: a `land` pass answers `pending` and a parked poll `awaiting-merge`, each with its next poll no earlier than
`resume_at`; the annotation sweep skips its pass; the close drain defers the intent to `resume_at` without spending its
failure backoff (`hub/domain/work_items/closure.py`); the garden's commit check answers that the forge could not say, as
it does for an unavailable one.

## What an operator sees

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `forge-budget-low` (warning), recorded once per
budget per reset window the first time a caller is turned away, and `forge-rate-limited` (warning), recorded once per
window when the forge itself refuses. Each names the forge, the secret, the remaining and limit it last read, and the
reset. `tests/event_log_kind_census.py` and `blizzard-context:/domain/operations.md` gain both. A forge's document on
`GET /api/forges/{forge}` carries each of its budgets' last reading, so the admin screen's forge detail shows how close
each key runs to its limit.
