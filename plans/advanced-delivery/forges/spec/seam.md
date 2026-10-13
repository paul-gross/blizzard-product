# Landing seam

## The roles

The hub acts on a repository's forge through narrow Protocols in `hub/forge/landing.py`, one per line along which its
callers actually call, and never composed into one wide seam (`bzh:seam-size-ceiling`). An adapter is built for one
repository — its forge's kind and addresses, the repository's `owner/repo`, and its revealed credential — so no
operation names a forge, an owner, or a token. Every operation is `async`.

**`IForgeRefs`** — branches, revisions, and commits:

| Operation                 | Answers                                                                                       |
| ------------------------- | --------------------------------------------------------------------------------------------- |
| `branch_tip(branch)`      | The revision a branch points at, or `None` when it does not exist                             |
| `compare(base, head)`     | `Comparison(status, ahead, behind, merge_base, commits, files, truncated)`                    |
| `commit_exists(revision)` | `CommitPresence(exists, authored_at)`; `ForgeUnavailable` when the forge could not say either |

**`IForgeChanges`** — the changes, pull or merge requests, that carry work into a branch:

| Operation                                       | Answers                                                                                                  |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `changes(state, base, head=None)`               | The changes into `base`, `open` or `closed` — `closed` includes merged — newest first, bounded and paged |
| `open_change(head, base, title, body)`          | The new `Change`                                                                                         |
| `change(number)`                                | The `Change`                                                                                             |
| `update_from_base(number, expected_head)`       | `Update(outcome, method)`                                                                                |
| `merge(number, expected_head, method, message)` | The landed revision; on a change already merged, its `merged_revision` rather than an error              |

**`IForgeChecks`** — the checks a forge runs on a revision:

| Operation         | Answers                                                                           |
| ----------------- | --------------------------------------------------------------------------------- |
| `checks(ref)`     | `Check` for each check on `ref`, a revision or a branch whose tip the forge reads |
| `rerun_check(id)` | Whether the forge accepted the re-run                                             |

Later slices grow these roles rather than adding a wide one: `advance_ref` and `merge_into` join `IForgeRefs` and
`change_checks` joins `IForgeChecks` with the `policies` slice; `create_ref` and `delete_ref` join `IForgeRefs`,
`close_change` joins `IForgeChanges`, and a fourth role, `IForgeQueue`, arrives with the `trains` slice. Each stays
under twelve operations, and a consumer is typed against the roles it calls: `land` takes all three, the garden's commit
check only `IForgeRefs`.

## What crosses the seam

A `Comparison` carries everything the `land` action's head gate reads from a compare today, so `gate_head` and
`base_merge.py` port without losing their proof: `status` is `identical`, `ahead`, `behind`, or `diverged`; `commits`
are the head side's commits as `CommitSummary(revision, parents)`, which the gate walks along first parents; `files` are
`FileChange(path, previous_path, status, blob, patch, additions, deletions)`; `merge_base` is the merge base's revision;
and `truncated` says the forge cut the file list short, which `base_merge.py` refuses to prove across, as it refuses
GitHub's three-hundred-file cap today.

A `Change` is `Change(number, head, head_revision, base, state, mergeability, web_url, merged_revision)`. `state` is
`open`, `merged`, or `closed`. `mergeability` is `mergeable`, `behind`, `conflicting`, `blocked`, or `pending` — the
last while the forge has not yet decided. `merged_revision` is the commit the merge put on the base, set once `state` is
`merged`: a merge commit or a squash commit, never the change's head, so it is the only way the hub learns what a merge
it did not perform landed, and what an interrupted merge of its own landed.

An `Update` is `Update(outcome, method)`: `outcome` is `accepted`, `conflict` when the forge could not bring the change
up to its base without a person, or `refused` for any other refusal; `method` is `merge` or `rebase`, how the forge
brought it up.

A `Check` is `Check(id, name, outcome, detail, details_url)`. `outcome` is `pending`, `passed`, `failed`, or `neutral`,
and only `outcome` is ever routed on. `detail` is the forge's own word for the result, and `details_url` the page where
the check's log can be read; both are carried for display, so the findings a failed landing writes still name each
failing check with the link a worker follows to its log. `method` on `merge` is `merge`, `squash`, or `rebase`.

Each adapter maps its forge's vocabulary into these words and nothing else crosses the seam.

## The GitHub vocabulary

The GitHub adapter maps a pull request's `mergeable_state`:

| GitHub      | Seam          | What `land` does with it                                                         |
| ----------- | ------------- | -------------------------------------------------------------------------------- |
| `clean`     | `mergeable`   | Reads the checks and merges once they are green                                  |
| `unstable`  | `mergeable`   | The same: only checks no rule requires are failing or running                    |
| `has_hooks` | `mergeable`   | The same: GitHub Enterprise Server's word for mergeable with pre-receive hooks   |
| `blocked`   | `blocked`     | Reads the checks and merges once they are green; a merge the forge refuses waits |
| `behind`    | `behind`      | Brings the change up to its base                                                 |
| `dirty`     | `conflicting` | Routes `conflict`                                                                |
| `unknown`   | `pending`     | Waits                                                                            |
| `draft`     | `blocked`     | Waits; the hub never opens a draft, so only a person makes one                   |

A pull request's `state` is `open`, or `closed` with `merged_at` set, which is `merged`, or `closed` without it, which
is `closed`.

A check run's `status` and `conclusion` fold into one `outcome`:

| GitHub                                                                    | Seam      |
| ------------------------------------------------------------------------- | --------- |
| `status` `queued`, `in_progress`, `waiting`, `requested`, or `pending`    | `pending` |
| `conclusion` `cancelled` or `stale`                                       | `pending` |
| `conclusion` `success`                                                    | `passed`  |
| `conclusion` `neutral` or `skipped`                                       | `neutral` |
| `conclusion` `failure`, `timed_out`, `action_required`, `startup_failure` | `failed`  |

`neutral` counts as green wherever `passed` does, as `neutral` and `skipped` do today. A cancelled run waits rather than
counting against the change, as it does today. `update_from_base` answers `method` `merge`, which is what GitHub's
`update-branch` does. The routing and check tables `land_pr_ci` carries today, `_RouteTable` and `_CheckTable`, become
the adapter's unit tests, so every row today's script routes is pinned to the seam word that carries it.

## Failures and retries

A failure is one of four typed errors:

- `ForgeNotFound` — the repository, change, or check does not exist.
- `ForgeRejected(reason)` — a refusal the forge explained. `reason` is `auth`, `conflict`, `stale`, `protected`, or the
  forge's own message.
- `ForgeUnavailable` — no answer the hub can act on: a refused connection, a timeout, a 5xx.
- `ForgeRateLimited(resume_at)` — a `ForgeUnavailable` whose answer says when to come back.

Each adapter classifies its forge's answers into these. The GitHub adapter does so per answer:

| GitHub answers                                                                                         | Error                                 |
| ------------------------------------------------------------------------------------------------------ | ------------------------------------- |
| A refused connection, a timeout, or any 5xx                                                            | `ForgeUnavailable`                    |
| A 403 or 429 with `X-RateLimit-Remaining: 0`, a `Retry-After`, or a body naming a secondary rate limit | `ForgeRateLimited(resume_at)`         |
| A 401, or a 403 that is not a rate limit                                                               | `ForgeRejected("auth")`               |
| A 404 on the repository                                                                                | `ForgeNotFound`                       |
| A 404 on a branch                                                                                      | No error: `branch_tip` answers `None` |
| A 422 from `update-branch` whose message names a conflict                                              | No error: `Update(conflict, merge)`   |
| Any other 422 from `update-branch`                                                                     | No error: `Update(refused, merge)`    |
| A 405 or 409 from `merge`                                                                              | `ForgeRejected(<message>)`            |
| Any other 4xx                                                                                          | `ForgeRejected(<message>)`            |

`resume_at` is the `Retry-After` when one is given, else the `X-RateLimit-Reset`, else a minute away; a secondary limit
is never resumed sooner than a minute. The adapter retries a `ForgeUnavailable` that is not rate-limited twice within
the call, after one second and then three, and raises it only when the third attempt fails too. It never retries a
`ForgeRateLimited` or a `ForgeRejected`. What each answer means for a landing — waiting, routing, or a person — is the
action's to decide ([landing.md](./landing.md#what-the-forge-answers)).

Write calls — opening, updating, merging, re-running — go out one at a time per credential: `ForgeRegistry` holds a lock
per forge and secret name, so a fleet finishing at once never bursts the forge's limits on content creation.

## Capabilities

A capability is a stable string naming one thing an adapter can do: `refs.read`, `compare`, `commits.read`,
`changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, `changes.merge.squash`,
`changes.merge.rebase`, `checks.read`, and `checks.rerun`. The set is closed and lives with the roles in
`hub/forge/landing.py`, beside `OPERATION_CAPABILITY`, which names the capability each operation needs — `merge` the one
its `method` names. Each adapter declares the capabilities it offers as a class constant. The role handles the registry
hands out check every call against `OPERATION_CAPABILITY` and the repository's offered set, so an operation whose
capability is not offered is refused with `ForgeCapabilityMissing(capability)` before it reaches the adapter.

`ForgeRegistry.offered(repository)` is the one place a caller asks what a repository's forge offers. In this slice it
answers the kind's set; a capability that varies by server or by repository narrows it in the slice that introduces it,
so it answers for a repository rather than a kind from the start.

## The GitHub adapter

`GitHubLanding` (`hub/forge/internal/github_landing.py`) implements all three roles and offers every capability. It is
the `land` action's GitHub client (`hub/delivery/actions/internal/github_client.py`), moved behind the seam: the same
routes — `git/ref/heads`, `compare`, `pulls`, `pulls/{n}`, `pulls/{n}/update-branch`, `pulls/{n}/merge`,
`commits/{ref}`, `commits/{ref}/check-runs`, `check-runs/{id}/rerequest` — the same paging bounds, and the same
`blizzard-hub` user on writes. Two reads the scripts leave at GitHub's first page of thirty, the open pull requests into
a base and the check runs on a revision, are paged to the same bound as the closed ones, so a busy repository never
hides the change or the failing check it is looking for. The open-change read now filters by base as the closed one
does, where today's script lists every open pull request; the adapter keeps filtering by base and head itself as well,
so a forge that ignores those parameters, as the mock forge does, answers the same. It uses `httpx.AsyncClient` with
`Authorization: token <secret>`, as the work-source adapter uses its synchronous client.

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
each declared to return exactly its role, and `offered(repository)`. Building reads the store and reveals a secret, so
the registry is the driver's to call, never an action's. For each repository a visit resolves, the driver bundles the
handles into a frozen `RepositoryForge(repository_id, refs, changes, checks, offered)` and hands it to the action beside
the `Visit`, credential already bound inside. Adapters are used only on the action thread's event loop, where every
action runs; `ConfigObjectCache` closes an object that carries `aclose` on that loop rather than calling a synchronous
`close`.

## The readers it absorbs

The garden's commit check stops being a separate forge path. The `garden-deliver` action receives a `RepositoryForge`
for each repository its chunk's project links, resolves each commit its delivery names to one of them through
`resolve_repository`, and asks that repository's `IForgeRefs.commit_exists` before it hands the delivery back. The
delivery carries the answers — a `ForgeUnavailable`, or a repository no link resolves, answers `None`, which degrades
the commit to a well-formedness check as it does today. `GardenDeliveryService` takes those answers in place of
`resolve_commit_for`, and the `CommitResolver` it hands the garden validation looks each pair up in them, so the
callable the validation takes is unchanged and makes no call of its own. `GitHubCommitResolver`
(`hub/forge/internal/commit_resolver.py`), `ICommitForge`, and `RepositoryCommitResolver`
(`hub/forge/repository_commits.py`) are deleted. `hub/forge/web.py::branch_url` reads the repository's forge for its web
origin.

The work-source family stays its own seam (`IWorkSource`, `IWorkAnnotator`, `IWorkCloser`), synchronous as it is. Only
its addresses move: the work-source factory (`hub/work_sources/internal/factory.py`) reads `api_base` and `web_base`
from the source's forge instead of its own record. Login (`IOAuthProvider`) names no forge; it is the hub's own
configuration and unchanged.
