# One rate budget per key

## Why now

Nothing in the hub reads a forge's rate limit today, and the call that spends it fastest is landing itself. A
merge-on-checks pass on a typical repository costs four or five calls: the first page of closed pull requests into the
base, to learn whether the branch already merged, the open ones listed, the pull request read, the head's check runs,
and a compare when the head has moved. On a repository busy enough to hold a hundred closed pull requests into its base,
the closed list is paged up to ten deep (`land_common.py`), and a pass costs up to fourteen. The shipped graphs poll
every thirty seconds, so a single delivering repository spends five or six hundred calls an hour, and up to 1,700 on a
busy one: ten landing at once, or three on busy repositories, exhaust a GitHub key's 5,000. Parked changes polled for
days, and the forge-status sweep and close drain sharing a credential with landing, add to that. This is the epic where
the hub must make a pass cheap, and then slow itself down before the forge stops it.

## The cost of a pass

Most of a pass's calls ask a question whose answer has not changed since the last pass, so the hub stops paying for
them:

- **Unchanged answers are free.** The GitHub adapter keeps each `GET`'s `ETag` per URL, in memory per adapter, and sends
  it back as `If-None-Match`; a `304 Not Modified` answers from what it kept, and GitHub does not count a 304 against
  the key's limit when the request is authenticated. This is where nearly all of the saving comes from. A work source
  that calls the forge with no secret is anonymous, and its 304s still count.
- **What cannot change is read once.** A compare between two fixed revisions is kept per adapter, so the head gate reads
  it once per pair rather than once per pass. An adapter lives as long as its repository's record goes unedited, and a
  compare carries its files' patches, so both stores are bounded, least recently used first out: 512 kept answers per
  adapter, and 64 compares. Checks are not kept the same way, however complete they look: a forge creates check runs as
  each workflow starts, so a revision whose every check has finished can still gain one, and a green read kept forever
  could merge ahead of a slower workflow. Their re-reads are answered 304 when nothing moved, which costs nothing
  anyway.
- **The already-merged lookup becomes one call.** Whether the chunk's branch already merged is
  `changes(closed, base, head=<owner>:<branch>)`, a filtered read, in place of paging through every closed pull request
  into the base. On most repositories that was one page already; on a busy one it was ten.
- **A pass with nothing new costs almost nothing.** With the reads above, a pass in which neither the head nor the base
  moved and the checks are still running is answered almost wholly by 304s.
- **The shipped graphs poll every minute.** Their delivery nodes' `poll_interval` goes from 30 to 60 seconds; a CI run
  takes minutes, and a minute's cadence lands within a minute of green.

The arithmetic, for one key on GitHub: a delivering repository in its steady state costs a counted call a pass at most,
sixty an hour, and usually fewer, and a pass where something moved — a new head, a moved base, a finished check — costs
four or five. A chunk's handful of writes (opening the change, an update, a re-run, the merge) count once each. A parked
change reads its repository's recently-closed listing every two minutes, and every parked change on the repository reads
the same address, so the listing is counted only when something into the base has closed since the last read — the first
reader after it pays, and the rest are answered 304 — however many changes are parked on it. So a key comfortably
carries several dozen repositories delivering at once, where today it carries ten.

## The budget

`ForgeBudgets` (`hub/forge/budget.py`) is held per tenant beside `ForgeRegistry` and keyed by forge, credential, and the
forge's own rate bucket — `(forge_id, secret_name, resource)` — where `resource` is GitHub's `X-RateLimit-Resource`,
`core` for every call the landing seam makes, and `core` for a forge that keeps one bucket. Keying by bucket keeps a
call the forge counts elsewhere — GitHub's GraphQL, which the trains epic's merge-queue reads speak, or its search —
from overwriting what the hub knows of the core budget. A work source with no secret calls the forge anonymously, and
its key's `secret_name` is null: every anonymous caller on that forge shares the one budget the forge gives the hub's
address. A forge counts what each user spends, and two secrets holding the same user's tokens share a limit the hub sees
twice; the budget reads the forge's own headers rather than counting, so each reading is still the truth about that
limit, and the hub only learns of a shared spend one answer late.

Every caller that holds a key on a forge shares that key's budget: the landing seam's adapters, the parked-change
listing, the garden's commit check, and the work-source family (`IWorkSource`, `IWorkAnnotator`, `IWorkCloser`,
`IWorkEditor`), whose adapters `StoreWorkSourceRegistry` now builds with their forge's budget.

A budget learns from the forge, never from a count of its own. Every response's rate headers update it — GitHub's
`X-RateLimit-Limit`, `-Remaining`, `-Reset`, and `-Resource` — through an `httpx` event hook every adapter's client
installs: an async hook on the landing seam's `httpx.AsyncClient`, which runs on the action loop, and a sync hook on the
work-source family's `httpx.Client`, which runs on sweep and request threads. Both call one
`ForgeBudgets.learn(key, headers)`, so a later forge kind's adapter teaches the budget from its own headers the same
way. The callers run on several threads at once, so each budget's reading is guarded by a `threading.Lock`, held only to
read or replace a handful of numbers and never across a request, so the action loop never waits on it for longer than
that. What counts as the forge refusing for rate is the forges epic's retry rules, which already raise
`ForgeRateLimited(resume_at)` for a spent remaining, a `Retry-After`, or a secondary limit; the budget hears of each one
and marks itself spent until `resume_at`. A budget the hub has not yet heard from admits every caller; it is held in
memory, and a restarted hub relearns it from its first answer. A GitHub Enterprise server with rate limiting turned off
sends no rate headers at all, so its budgets never learn a limit and admit every caller; there is nothing to share, and
the forges epic's retry rules still meet whatever refusal the server does send.

Admission happens before the request, so it must know which bucket a call spends before the forge says. Each call
declares it: every operation on the landing seam and the work-source family declares `core`, and a later caller that
speaks another of the forge's buckets — the trains epic's GraphQL reads, a search — declares that one. The answer's
`X-RateLimit-Resource` then says which bucket it actually spent, and the reading goes to that bucket whichever was
declared.

## Who goes first

Each call carries how much it matters, and a budget running low turns the least important away first, before any request
is made. Priority belongs to the call, not to the pass: a `land` pass spends `landing` on the repositories the hub lands
itself and `watching` on a parked repository. Adapters are built and cached once per repository and shared by every
caller of it — a landing chunk and a parked one on the same repository read through one adapter — so the priority cannot
live on the adapter. It lives on the role handle instead: `ForgeRegistry.refs`, `changes`, and `checks` each take a
`priority`, and hand back a thin handle that carries it into every call on the shared adapter, where the budget reads
it. The driver asks for `landing` handles for the repositories the hub lands itself and `watching` handles for a parked
one, so a policy never names a priority; the garden's commit check asks `ForgeRegistry.refs` for a `sweeping` handle
directly, with no driver in between. The work-source family has no handle to carry one, and one client serves callers of
every kind, so each of its operations has a fixed priority of its own, which its adapter hands the budget with the call:
`IWorkEditor`'s operations are `landing`, being a person's own edit, and every operation of `IWorkSource`,
`IWorkAnnotator`, and `IWorkCloser` is `sweeping`.

| Priority   | Callers                                                                                                          | Admitted while the remaining budget is |
| ---------- | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `landing`  | A `land` pass for a repository the hub lands itself; a person's own edit of a work item through `IWorkEditor`    | above nothing — until spent            |
| `watching` | A parked change's reads, the recently-closed listing                                                             | above a fifth of the limit             |
| `sweeping` | The forge-status annotation sweep, the close drain, work-source ingest and resolution, the garden's commit check | above two fifths of the limit          |

A call turned away raises `ForgeBudgetLow(resume_at)` without reaching the forge. `ForgeBudgetLow` is a
`ForgeRateLimited`, so every landing caller that already waits out a rate limit waits it out the same way, and each
caller slows rather than fails:

- A `land` pass answers `pending`, or `awaiting-merge` while a parked change is open, with `Pass.next_poll_at` set to
  `resume_at`, so its next poll is no earlier.
- The work-source family does not import `hub/forge`, so its adapters raise their own family's error for it —
  `WorkSourceError`, `WorkAnnotateError`, or `WorkCloseError` — carrying `resume_at` as `retry_after`. The annotation
  sweep and ingest skip their pass; a person's edit answers `503` with `Retry-After`.
- The close drain defers the intent to `resume_at` without spending its failure backoff. `close_intent_attempts`
  (`hub/store/schema.py`) gains the outcome `deferred` beside `skipped` and `failed`, with a nullable `resume_at`
  column; `close_intent_is_due` (`hub/domain/work_items/closure.py`) leaves deferred rows out of the attempt count and
  the backoff it derives, and holds an intent not due until its newest deferred row's `resume_at` has passed.
- The garden's commit check answers that the forge could not say, as it does for an unavailable one.

A `land` pass that waits on the budget waits inside its node's poll timeout like any `pending`, and the three-hour
timeout is what keeps a reset an hour away from bouncing it.

## What an operator sees

`EventLogKind` and `EVENT_LOG_SEVERITY` (`foundation/event_log.py`) gain `forge-budget-low` (warning), recorded once per
budget per reset window the first time a caller is turned away, and `forge-rate-limited` (warning), recorded once per
window when the forge itself refuses. Each names the forge, the secret, the bucket, the remaining and limit it last
read, and the reset. A budget lives beside adapters that hold no store, so it never writes an event itself: it queues
each notice in memory, once per window, and the flow driver's tick drains the tenant's queue into `event_log` through
`EventLogService.record`, off the event loop as every driver write is. A notice queued just before the hub stops is lost
with it, and the next window records afresh. `tests/event_log_kind_census.py` and
`blizzard-context:/domain/operations.md` gain both. A forge's document on `GET /api/forges/{forge}` carries each of its
budgets' last reading, so the admin screen's forge detail shows how close each key runs to its limit.
