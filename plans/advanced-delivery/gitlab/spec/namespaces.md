# Where a GitLab repository lives

## Nested groups

GitLab groups nest, so a GitLab project's owner is a path (`group/subgroup`). A GitLab repository record's `owner` holds
the full namespace, and `validate_fields` (`hub/domain/config/repositories.py`) admits `/` in `owner` when the record's
forge is GitLab, and refuses it on a GitHub repository, whose owner is always one segment.

`RepoRef` (`foundation/repo_ref.py`, its cases in `tests/test_repo_ref.py`) parses a nested origin's owner as its
immediate parent today. It gains `namespace`, every segment between host and name, and `CommitOrigin` carries it, so
`_candidates` matches a GitLab record's owner against the origin's whole namespace and a GitHub record's against its one
segment, as now. `repo_identity` and `pointer_key` in the same module key by the whole namespace too, so `a/x/r` and
`b/x/r` on one host stay two repositories.

Both functions run on the runner (`runner/lifecycle/judgement/git_commits.py`) as well as at the hub, so the runner and
hub ship the change in one release ([carry-over.md](./carry-over.md#shipping)). A GitHub origin, whose namespace is its
one owner, keys exactly as it does today, so no stored pointer or marker written before the slice changes meaning.

## GitLab's web addresses

`hub/forge/web.py` learns GitLab's web shapes — `/-/tree/<branch>`, `/-/commit/<sha>`, `/-/merge_requests/<iid>`, and a
web origin derived from `api_base` by dropping `/api/v4` — chosen by the forge's `kind`. `delivery_read.py`'s
`_commit_url`, which today parses a GitHub pull request's address, asks `web.py` for a commit's address from the
repository's forge instead, so the board links a GitLab chunk's commits and merge request as it links a GitHub one's.
