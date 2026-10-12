# Landing seam

## The Protocol

`IForgeLanding` (`hub/forge/landing.py`) is the one way the hub acts on a repository's forge. An adapter is built for
one repository — its forge's kind and addresses, the repository's `owner/repo`, and its revealed credential — so no
operation names a forge, an owner, or a token:

| Operation                                       | Answers                                                                                               |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `branch_tip(branch)`                            | The revision a branch points at, or `None` when it does not exist                                     |
| `compare(base, head)`                           | `Comparison(ahead, behind, status)` — `identical`, `ahead`, `behind`, or `diverged`                   |
| `commit_exists(revision)`                       | `CommitPresence(exists, authored_at)`, or `None` when the forge could not say                         |
| `changes(state, base, head=None)`               | The changes into `base`, `open` or `closed`, newest first, bounded as `land_common` bounds them today |
| `open_change(head, base, title, body)`          | The new `Change`                                                                                      |
| `change(number)`                                | `Change(number, head, head_revision, base, state, mergeability, web_url)`                             |
| `update_from_base(number, expected_head)`       | Whether the forge accepted bringing the change up to its base                                         |
| `merge(number, expected_head, method, message)` | The landed revision                                                                                   |
| `checks(revision)`                              | `Check(id, name, outcome)` for each check on the revision                                             |
| `rerun_check(id)`                               | Whether the forge accepted the re-run                                                                 |

A `Change` carries the seam's own words. `state` is `open`, `merged`, or `closed`. `mergeability` is `mergeable`,
`behind`, `conflicting`, `blocked`, or `pending` — the last while the forge has not yet decided. A check's `outcome` is
`pending`, `passed`, `failed`, or `neutral`. `method` is `merge`, `squash`, or `rebase`. Each adapter maps its forge's
vocabulary into these and nothing else crosses the seam: the GitHub adapter turns `mergeable_state` `clean` into
`mergeable` and `dirty` into `conflicting`, and folds a check run's `status` and `conclusion` into one `outcome`, so
`land_pr_ci.py`'s `Verdict.of` reads `outcome` where it reads GitHub's fields today.

Failures are three typed errors: `ForgeNotFound`, `ForgeRejected(reason)` for a refusal the forge explained, and
`ForgeUnavailable` for a 5xx, a timeout, or no answer. None is swallowed inside the adapter.

## Capabilities

A capability is a stable string naming one thing an adapter can do: `refs.read`, `compare`, `commits.read`,
`changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, `changes.merge.squash`,
`changes.merge.rebase`, `checks.read`, and `checks.rerun`. The set is closed and lives with the Protocol. Each adapter
declares the capabilities it offers as a class constant, and an operation whose capability the adapter does not offer is
never called: the registry refuses it with `ForgeCapabilityMissing(capability)` before it reaches the adapter.

A forge's capabilities are its kind's in this slice. A capability that varies by server or by repository narrows this
set in the slice that introduces it, so `offered(repository)` is the one place a caller asks, and it answers for a
repository rather than a kind from the start.

## The GitHub adapter

`GitHubLanding` (`hub/forge/internal/github_landing.py`) implements every operation and offers every capability. It is
the forge-calling half of `hub/graphs/scripts/land_common.py`, moved behind the seam: the same routes — `git/ref/heads`,
`compare`, `pulls`, `pulls/{n}`, `pulls/{n}/update-branch`, `pulls/{n}/merge`, `commits/{ref}`,
`commits/{ref}/check-runs`, `check-runs/{id}/rerequest` — the same paging bounds, and the same `blizzard-hub` user on
writes. It uses `httpx` with `Authorization: token <secret>`, as the work-source adapter does.

## The registry

`ForgeRegistry` (`hub/forge/registry.py`) builds an adapter for a repository record on use: it reads the record and its
forge, reveals the repository's secret through `ISecretReader`, and picks the adapter class by `kind`. Built adapters
are cached in a `ConfigObjectCache` keyed by the repository's and the forge's revisions, so an edit to either takes
effect on the next use, as `StoreWorkSourceRegistry` does for work sources. It is wired per tenant in `hub/app.py`
beside `work_source_registry`.

## The readers it absorbs

The garden's commit check stops being a separate forge path. `RepositoryCommitResolver`
(`hub/forge/repository_commits.py`) asks `ForgeRegistry` for the resolved repository's adapter and calls
`commit_exists`, and `GitHubCommitResolver` (`hub/forge/internal/commit_resolver.py`) is deleted; the `CommitResolver`
callable the garden validation takes is unchanged. `hub/forge/web.py::branch_url` reads the repository's forge for its
web origin.

The work-source family stays its own seam (`IWorkSource`, `IWorkAnnotator`, `IWorkCloser`). Only its addresses move: the
work-source factory (`hub/work_sources/internal/factory.py`) reads `api_base` and `web_base` from the source's forge
instead of its own record. Login (`IOAuthProvider`) names no forge; it is the hub's own configuration and unchanged.
