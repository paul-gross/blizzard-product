# Landing seam

## The roles

The hub acts on a repository's forge through narrow Protocols in `hub/forge/landing.py`, one per line along which its
callers actually call, and never composed into one wide seam (`bzh:seam-size-ceiling`). The module holds the Protocols
and the value types that cross them, and nothing that imports `httpx`, so an action may import it under the actions
epic's purity allowlist. An adapter is built for one repository — its forge's kind and addresses, the repository's
`owner/repo`, and its revealed credential — so no operation names a forge, an owner, or a token. Every operation is
`async`.

**`IForgeRefs`** — branches, revisions, and commits:

| Operation                 | Answers                                                                                             |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| `branch_tip(branch)`      | The revision a branch points at, or `None` when it does not exist                                   |
| `compare(base, head)`     | `Comparison(status, ahead, behind, merge_base, commits, commits_truncated, files, files_truncated)` |
| `commit_exists(revision)` | `CommitPresence(exists, authored_at)`; `ForgeUnavailable` when the forge could not say either       |

**`IForgeChanges`** — the changes, pull or merge requests, that carry work into a branch:

| Operation                                       | Answers                                                                                                  |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `changes(state, base, head=None)`               | The changes into `base`, `open` or `closed` — `closed` includes merged — newest first, bounded and paged |
| `open_change(head, base, title, body)`          | The new `Change`                                                                                         |
| `change(number)`                                | The `Change`                                                                                             |
| `update_from_base(number, expected_head)`       | `Update(outcome, method)`                                                                                |
| `merge(number, expected_head, method, message)` | The landed revision                                                                                      |

**`IForgeChecks`** — the checks a forge runs on a revision:

| Operation         | Answers                                                                           |
| ----------------- | --------------------------------------------------------------------------------- |
| `checks(ref)`     | `Check` for each check on `ref`, a revision or a branch whose tip the forge reads |
| `rerun_check(id)` | Whether the forge accepted the re-run                                             |

`ForgeRegistry` bundles the three roles for one repository into a frozen
`RepositoryForge(repository_id, refs, changes, checks, offered)`, and `VisitForges`, the mapping the actions epic hands
`land`, is declared beside it from this epic on as `Mapping[RepositoryId, RepositoryForge]`, so both live in
`hub/forge/landing.py` with the roles they carry ([The registry](#the-registry)).

Later epics grow these roles rather than adding a wide one: with the `policies` epic, `advance_ref`, `merge_into`,
`create_ref`, and `delete_ref` join `IForgeRefs`, `recently_closed` joins `IForgeChanges`, and `change_checks` joins
`IForgeChecks`; with the `trains` epic, `close_change` joins `IForgeChanges` and a fourth role, `IForgeQueue`, arrives.
Each stays under twelve operations, and a consumer is typed against the roles it calls: `land` takes all three, the
garden's commit check only `IForgeRefs`.

## What crosses the seam

A `Comparison` carries everything the `land` action's head gate reads from a compare today, so `gate_head` and
`base_merge.py` port without losing their proof: `status` is `identical`, `ahead`, `behind`, or `diverged`; `commits`
are the head side's commits as `CommitSummary(revision, parents)`, which the gate walks along first parents; `files` are
`FileChange(path, previous_path, status, blob, patch, additions, deletions)`; and `merge_base` is the merge base's
revision. The two truncation flags are kept apart because their readers are: `commits_truncated` says the forge returned
fewer commits than the head is `ahead`, which the head gate refuses to admit across, as `gate_head` refuses a commit
list shorter than GitHub's `total_commits` today; `files_truncated` says the forge cut the file list short, which
`base_merge.py` refuses to prove across, as it refuses GitHub's three-hundred-file cap today.

A `Change` is `Change(number, head, head_revision, base, base_revision, state, mergeability, web_url, merged_revision)`.
`base_revision` is the base's revision the forge last compared the change against — GitHub's `base.sha` — which `land`
does not read in this epic but a second forge kind gates on, so it crosses the seam from the start. `state` is `open`,
`merged`, or `closed`. `mergeability` is `mergeable`, `behind`, `conflicting`, `blocked`, `draft`, or `pending` — the
last while the forge has not yet decided, or answers a state the hub does not merge through. `merged_revision` is the
commit the merge put on the base, set once `state` is `merged`: a merge commit or a squash commit, never the change's
head, so it is the only way the hub learns what a merge it did not perform landed, and what an interrupted merge of its
own landed.

An `Update` is `Update(outcome, method)`: `outcome` is `accepted`, `conflict` when the forge could not bring the change
up to its base without a person, or `refused` for any other refusal; `method` is `merge` or `rebase`, how the forge
brought it up.

A `Check` is `Check(id, name, outcome, detail, details_url)`. `outcome` is `pending`, `passed`, `failed`, or `neutral`,
and only `outcome` is ever routed on. `detail` is the forge's own word for the result, and `details_url` the page where
the check's log can be read; both are carried for display, so the findings a failed landing writes still name each
failing check with the link a worker follows to its log. `method` on `merge` is `merge`, `squash`, or `rebase`.

Each adapter maps its forge's vocabulary into these words and nothing else crosses the seam.

## The GitHub vocabulary

The GitHub adapter maps a pull request's `mergeable_state` so that `land` decides exactly what `land_pr_ci`'s `Route`
decides today: only `clean`, `unstable`, and `blocked` are states whose checks it reads before merging
(`_CI_WATCH_STATES`), and every state it does not name waits.

| GitHub             | Seam          | What `land` does with it                                                         |
| ------------------ | ------------- | -------------------------------------------------------------------------------- |
| `clean`            | `mergeable`   | Reads the checks and merges once they are green                                  |
| `unstable`         | `mergeable`   | The same: only checks no rule requires are failing or running                    |
| `blocked`          | `blocked`     | Reads the checks and merges once they are green; a merge the forge refuses waits |
| `behind`           | `behind`      | Brings the change up to its base                                                 |
| `dirty`            | `conflicting` | Routes `conflict`                                                                |
| `draft`            | `draft`       | Waits; the hub never opens a draft, so only a person makes one, or undoes it     |
| `has_hooks`        | `pending`     | Waits, as today: the hub has never merged through a pre-receive hook state       |
| `unknown`          | `pending`     | Waits                                                                            |
| Any other, or none | `pending`     | Waits                                                                            |

A pull request's `state` is `open`, or `closed` with `merged_at` set, which is `merged`, or `closed` without it, which
is `closed`.

A check run's `status` and `conclusion` fold into one `outcome`. A run is green only when it has completed with
`success`, `neutral`, or `skipped`, and red only when it has completed with one of the three conclusions `land_pr_ci`
treats as terminal today (`_TERMINAL_CONCLUSIONS`); everything else waits:

| GitHub                                                                       | Seam      |
| ---------------------------------------------------------------------------- | --------- |
| `status` `completed`, `conclusion` `success`                                 | `passed`  |
| `status` `completed`, `conclusion` `neutral` or `skipped`                    | `neutral` |
| `status` `completed`, `conclusion` `failure`, `timed_out`, `action_required` | `failed`  |
| `status` `completed`, `conclusion` `cancelled`, `stale`, `startup_failure`   | `pending` |
| `status` `completed`, any other conclusion, or none                          | `pending` |
| Any other `status` — `queued`, `in_progress`, `waiting`, `requested`, …      | `pending` |

`neutral` counts as green wherever `passed` does, as `neutral` and `skipped` do today. A cancelled run waits rather than
counting against the change, as it does today, and so does a run that failed to start, which no re-poll of a check run's
payload can tell from one that will be restarted. A read with no checks at all is never green, as `Verdict.green` holds
today. `update_from_base` answers `method` `merge`, which is what GitHub's `update-branch` does. The routing and check
tables `land_pr_ci` carries today, `_RouteTable` and `_CheckTable`, become the adapter's unit tests, so every row
today's script routes is pinned to the seam word that carries it.

## Failures and retries

A failure is one of four typed errors:

- `ForgeNotFound(what)` — something the call named does not exist: `what` is `repository`, `branch`, `change`, `check`,
  or `revision`.
- `ForgeRejected(reason)` — a refusal the forge explained. `reason` is `auth`, `conflict`, `stale`, `protected`, or the
  forge's own message.
- `ForgeUnavailable` — no answer the hub can act on: a refused connection, a timeout, a 5xx.
- `ForgeRateLimited(resume_at)` — a `ForgeUnavailable` whose answer says when to come back.

Each adapter classifies its forge's answers into these. The GitHub adapter does so per answer:

| GitHub answers                                                                                         | Error                                   |
| ------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| A refused connection, a timeout, or any 5xx                                                            | `ForgeUnavailable`                      |
| A 403 or 429 with `X-RateLimit-Remaining: 0`, a `Retry-After`, or a body naming a secondary rate limit | `ForgeRateLimited(resume_at)`           |
| A 401 or a 403 from `rerun_check` that is not a rate limit                                             | No error: `rerun_check` answers refused |
| A 401, or a 403 that is not a rate limit, from any other operation                                     | `ForgeRejected("auth")`                 |
| A 404 from `branch_tip`, the repository present                                                        | No error: `branch_tip` answers `None`   |
| A 404 from `commit_exists`, the repository present                                                     | No error: `CommitPresence(False)`       |
| Any other 404                                                                                          | `ForgeNotFound(what)`, below            |
| A 422 from `update-branch` whose message names a conflict                                              | No error: `Update(conflict, merge)`     |
| Any other 4xx from `update-branch`, save a 401 or 403                                                  | No error: `Update(refused, merge)`      |
| A 405 or 409 from `merge`, including one on a change already merged                                    | `ForgeRejected(<message>)`              |
| Any other 4xx                                                                                          | `ForgeRejected(<message>)`              |

GitHub answers a bare 404 alike for a repository that does not exist, for one the credential cannot see, and for a pull
request, a check run, or a revision that is not there, so the status alone cannot say which. On a 404 the adapter reads
the repository itself, `GET /repos/{owner}/{repo}`, once: a 404 there too is `ForgeNotFound("repository")`, and
otherwise the 404 names the thing the operation asked for — `branch_tip` answers `None`, `commit_exists` answers
`CommitPresence(False)`, and the rest raise `change` from `change`, `merge`, and `update-branch`, `check` from
`rerun_check`, `revision` from `compare` and `checks`. A missing base branch and a missing repository therefore part at
the first pass rather than both waiting out the poll timeout as an unread tip. The adapter keeps the probe's answer,
either way, for sixty seconds, so a run of misses within a pass costs one extra read while a repository created,
deleted, or re-shared later is seen within a minute; an adapter lives in the tenant's cache until its records are
edited, so remembering the answer for the adapter's life would remember it for days. Because GitHub hides a repository
from a credential that cannot see it, a `ForgeNotFound("repository")` always says both: the repository does not exist,
or this credential cannot reach it.

The one 401 or 403 that is not a rejected credential is the re-run's. GitHub grants a check-run re-request only to a
GitHub App, so the token a repository lands with is refused there as a matter of course while it remains right for every
other call; `land` treats a refused re-run as no re-run, as the script does today when `_rerequest_once` ignores any
answer but a 2xx. `rerun_check` answers refused for it, never `ForgeRejected("auth")`.

`ForgeRejected`'s named reasons `conflict`, `stale`, and `protected` are the seam's vocabulary for the ref writes the
`policies` epic adds; no answer to an operation this epic builds maps to them.

`resume_at` is the `Retry-After` when one is given, else the `X-RateLimit-Reset`, else a minute away; a reset already in
the past, or zero, counts as none given, and a secondary limit is never resumed sooner than a minute. Every attempt
carries a thirty-second timeout, the one the work-source client already uses, where today's script sets none. The
adapter retries a `ForgeUnavailable` that is not rate-limited twice within the call, after one second and then three,
and raises it only when the third attempt fails too, so a call never runs past about a minute and a half, well inside a
pass's five. It never retries a `ForgeRateLimited`, a `ForgeRejected`, or a `ForgeNotFound`. What each answer means for
a landing — waiting, routing, or a person — is the action's to decide, operation by operation
([landing.md](./landing.md#what-the-forge-answers)).

Write calls — opening, updating, merging, re-running — go out one at a time per credential, so a fleet finishing at once
never bursts the forge's limits on content creation. `ForgeRegistry` holds an `asyncio.Lock` per forge and secret name,
created on the action thread's loop where every adapter runs, and hands the same lock to every adapter it builds for
that pair. The lock is held around each attempt and released through the backoff between attempts, so one call waiting
out a flaky forge never holds every other write on that credential behind its sleep.

## Capabilities

A capability is a stable string naming one thing an adapter can do: `refs.read`, `compare`, `commits.read`,
`changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, `changes.merge.squash`,
`changes.merge.rebase`, `checks.read`, and `checks.rerun`. The set is closed, a `Capability` `StrEnum` in
`hub/domain/config/capabilities.py`, so the domain can reason about what a repository's forge offers —
`ACTION_REQUIREMENTS`, and later a policy's — without importing the seam. `FORGE_KIND_CAPABILITIES`
(`hub/domain/config/forges.py`), beside `KNOWN_FORGE_KINDS`, names what each kind offers. The seam imports the
vocabulary rather than defining it: `OPERATION_CAPABILITY` in `hub/forge/landing.py` names the capability each operation
needs — `merge` the one its `method` names. Each adapter declares the capabilities it offers as a class constant, and a
unit test holds that constant equal to its kind's `FORGE_KIND_CAPABILITIES` entry, so the domain's answer and the
adapter's never drift. The role handles the registry hands out check every call against `OPERATION_CAPABILITY` and the
repository's offered set, so an operation whose capability is not offered is refused with
`ForgeCapabilityMissing(capability)` before it reaches the adapter.

`ForgeRegistry.offered(forge, repository=None)` is the one place a caller asks what a forge offers, and for a repository
when one is named. In this epic it answers the kind's set either way; a capability that varies by server or by
repository narrows it in the epic that introduces it, so it takes a repository from the start, and takes it as optional
so a caller checking a setting for a repository not yet created — the `policies` epic's set-time check — asks the forge
alone. The registry is handed the kind-to-capabilities map at composition, `FORGE_KIND_CAPABILITIES` in production, so a
test hands it a narrower one and proves the refusal without a second adapter.

## The GitHub adapter

`GitHubLanding` (`hub/forge/internal/github_landing.py`) implements all three roles and offers every capability. It is
the client the actions epic gave `land` — `GitHubHandle` (`hub/delivery/internal/github_handle.py`) behind the
`IGitHubHandle` Protocol (`hub/delivery/forge_handle.py`) — moved behind the seam: the same routes — `git/ref/heads`,
`compare`, `pulls`, `pulls/{n}`, `pulls/{n}/update-branch`, `pulls/{n}/merge`, `commits/{ref}`,
`commits/{ref}/check-runs`, `check-runs/{id}/rerequest` — the same paging bounds, and the same `blizzard-hub` user on
writes, with `GET /repos/{owner}/{repo}` added for the 404 probe above. Two reads the scripts leave at GitHub's first
page of thirty, the open pull requests into a base and the check runs on a revision, are paged to the same bound as the
closed ones, so a busy repository never hides the change or the failing check it is looking for. The open-change read
now filters by base as the closed one does, where today's script lists every open pull request; the adapter keeps
filtering by base and head itself as well, so a forge that ignores those parameters, as the mock forge does, answers the
same. It uses `httpx.AsyncClient` with `Authorization: token <secret>`, as the work-source adapter uses its synchronous
client.

## The registry

`ForgeRegistry` (`hub/forge/registry.py`) builds an adapter for a repository record on use: it reads the record and its
forge, reveals the repository's secret through `ISecretReader`, and builds the adapter through the build callable the
composition root hands it, which `hub/forge/internal/factory.py` supplies by `kind` — so no adapter class is imported
outside the factory and the composition roots, as `work_sources/internal/factory.py` keeps its own. A repository or
forge retired after a chunk named it is still built for that chunk, as the work-source registry reaches retired records.
Built adapters are cached in the tenant's `ConfigObjectCache` under the repository's `CacheKey`, which carries the
forge's revision beside the repository's and its secret's ([record.md](./record.md#the-change-log)), so an edit to any
of the three takes effect on the next use, as `StoreWorkSourceRegistry` does for work sources. It is wired per tenant in
`hub/app.py` beside `work_source_registry`.

The registry hands out roles, never an adapter: `refs(repository)`, `changes(repository)`, and `checks(repository)`,
each declared to return exactly its role, and `offered(forge, repository=None)`. Building reads the store and reveals a
secret, so the registry is called by whoever builds a visit — `HubNodeExecutor` in this epic — and never by an action. A
secret that cannot be revealed raises the domain's `RepositorySecretUnavailable`, and the executor answers it with the
actions epic's refusal unchanged (`hub-log.repository-secret-unavailable`, routed `failure`). For each repository a
visit resolves, the executor bundles the handles into a `RepositoryForge` ([The roles](#the-roles)), and the
`VisitForges` mapping the actions epic hands `land` now carries a `RepositoryForge` per repository in place of an
`IGitHubHandle`, the credential already bound inside. `IGitHubHandle`, `GitHubHandle`, and
`hub/delivery/forge_handle.py` are deleted. The actions epic's purity allowlist trades
`blizzard.hub.delivery.forge_handle` for `blizzard.hub.forge.landing`, where the roles, their value types,
`RepositoryForge`, and `VisitForges` live, and gains `blizzard.hub.domain.config.capabilities`, since `land` names
`Capability.CHECKS_RERUN` when it asks what is offered; neither module imports `httpx`.

Adapters are used only on the action thread's event loop, where every action runs, and an `httpx.AsyncClient` must be
closed on the loop it ran on. `ConfigObjectCache` closes a superseded object synchronously, under its own lock, from
whichever thread read the edit (`hub/live_config/__init__.py`), and stays that way: each adapter implements the cache's
synchronous `close()` by scheduling its own `aclose()` onto the action loop through `loop.call_soon_threadsafe(...)` and
returning at once. A pass still holding the superseded adapter finishes its call first, since the cache closes an entry
one replacement late. At shutdown the hub closes each tenant's `ConfigObjectCache` before it stops the `ActionLoop`, so
every adapter's `aclose()` runs on a live loop; a `close()` that still finds the loop stopped logs and drops the client,
which the ending process releases anyway, rather than raising out of the cache.

## The readers it absorbs

The garden's commit check stops being a separate forge path, and `garden-deliver` stays the action the actions epic made
it: it holds no branch (`ActionSpec.lands` is false), is handed no forge, and reveals no secret. The check stays where
the actions epic left it, in the effect writer that runs `GardenDeliveryRecorder`
(`hub/domain/garden/delivery/service.py`) when the action hands back its `RecordGardenDelivery`, and only its resolver
changes. `SeamCommitResolver` (`hub/forge/commit_check.py`) replaces `RepositoryCommitResolver`
(`hub/forge/repository_commits.py`) with the same shape — `for_project(project_id)` answers the `CommitResolver`
callable the garden validation already takes, never raising. For each cited `(repo, commit)` pair it resolves the
repository through `resolve_repository` over the repositories the project links, as today, asks `ForgeRegistry.refs` for
that repository, and runs `commit_exists` on the action loop from the writer's thread through
`asyncio.run_coroutine_threadsafe`, waiting at most ten seconds and cancelling the future when the wait runs out, so an
abandoned check never keeps a forge call running on the action loop. A repository no link resolves, a secret that cannot
be revealed, a `ForgeUnavailable`, a `ForgeNotFound("repository")`, a timeout, or any other error answers `None`, which
degrades the commit to a well-formedness check as it does today; a `CommitPresence` answers what it says. One answer
changes on purpose: today a 404 on a repository the credential cannot see answers the commit absent
(`commit_resolver.py`), which refuses a delivery for a commit nobody could look for; through the seam's probe it is a
`ForgeNotFound("repository")` and degrades to `None`, as every other unreachable repository already does.
`GardenDeliveryRecorder` and the validation callable are unchanged. `GitHubCommitResolver`
(`hub/forge/internal/commit_resolver.py`), `ICommitForge`, and `RepositoryCommitResolver` are deleted.
`hub/forge/web.py::branch_url` reads the repository's forge for its effective web origin, which the domain derives
([record.md](./record.md#the-table)).

Only that reader moves onto the seam. The work-source family stays its own seam (`IWorkSource`, `IWorkAnnotator`,
`IWorkCloser`), synchronous as it is, on its own client: the GitHub work source's reads, labels, comments, and closes
(`hub/work_sources/internal/github_work_source.py`) and the forge-status annotation sweep that drives them
(`hub/domain/observability/forge_status.py`) keep speaking GitHub directly, and the `policies` epic brings them under
the same rate budget the seam's adapters share. Only their addresses move: the work-source factory
(`hub/work_sources/internal/factory.py`) reads `api_base` and `web_base` from the source's forge instead of its own
record. Login (`IOAuthProvider`) names no forge; it is the hub's own configuration and unchanged.
