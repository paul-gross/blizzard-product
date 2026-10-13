# Landing on GitLab

## The policies, unchanged

No policy changes for GitLab. `land` runs each repository's policy against the `RepositoryForge` it is handed
([policies' landing.md](../../policies/spec/landing.md)), and a GitLab repository's roles answer in the seam's words, so
`merge-on-checks`, `park`, the combining step, and the project's order read a GitLab repository exactly as they read a
GitHub one. What differs is what the forge offers and what it can say:

- **`merge-on-checks`** reads a change's checks through `change_checks`, which the GitLab adapter answers from the merge
  request's own pipeline ([adapter.md](./adapter.md#its-operations)), and brings a change up to its base with
  `update_from_base`, which GitLab answers by rebasing ([The head gate](#the-head-gate)).
- **`park`** finds a person's merge in the recently-closed listing, ordered by update, as on GitHub.
- **`fast-forward`** needs `refs.advance`, which no GitLab repository offers. Setting it on a GitLab repository is
  refused `422 forge_capability_missing` naming `refs.advance`, by the check the policies slice runs when a policy is
  set or a repository's forge is changed, so a fast-forwarding repository cannot be moved onto a GitLab forge either.

## The head gate

GitHub brings a change up to its base by merging the base in; GitLab rebases it, rewriting the head, so the head no
longer descends from the submitted commit and today's gate would call it foreign. GitLab's comparisons cannot prove a
rebase carried the work unchanged the way `base_merge.py` proves a base merge, because they carry no blob ids, no
per-file line counts, and no total that would show a commit list cut short. So a rebased head is admitted only on the
hub's own word, checked against what GitLab can say:

1. When `update_from_base` answers method `rebase`, `land` hands `rebase-requested/<repo>/<head>` to its effects, naming
   the head it rebased from, before it asks the forge to act.
2. `gate_head` (`hub/delivery/actions/landing/head_gate.py`) admits a later head that does not descend from the
   submitted commit only when such a marker names the head it replaced, the change's `diff_refs.base_sha` is the base's
   current tip, the rebased head carries as many commits past that base as the replaced head carried past its own, and
   the set of paths it changes is the set the replaced head changed.

A head that fails any of these — a rebase nobody asked the hub for, a person's push in between, a rebase that dropped or
added a file — is a foreign head, and `failure`, as today. The check trades the proof GitHub's base merges get for one
the forge can answer, and it stands on the narrow fact that the hub asked for this rebase and GitLab performed it.

The window between the marker and the request is a named point in the crash-point registry (`bzh:crash-point-registry`),
reached by a `blizzard:crash-sweep` scenario:

| Point                                       | Between                                                              | The next pass                                          |
| ------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------ |
| `policy.after-rebase-marker.before-request` | `rebase-requested/<repo>/<head>` is durable; no rebase was requested | Sees the head unchanged, and requests the rebase again |

## What each policy proves on GitLab

What the policies slice says each policy proves holds on GitLab
([policies' landing.md](../../policies/spec/landing.md#what-each-policy-proves)). A GitLab project merging fast-forward
or semi-linear refuses a merge request that is not up to date with its base, so there `merge-on-checks` merges only a
change whose checks ran on the base it lands on, and the forge closes the gap that a merge-commit project leaves open. A
project running merged-results pipelines proves the merge GitLab builds rather than the head, which narrows the gap
further without the hub doing anything different.

## Landing across kinds of forge

A chunk whose project links a GitHub repository and a GitLab one resolves each commit pointer to its own record, as
every chunk does, and its visit holds both repositories' flows. `combine.py` folds the two repositories' states into one
answer without knowing their kinds, and lands them in the project's order: neither until every repository the hub lands
itself is ready, then one at a time, a landed repository standing should a later one refuse. The two writes are on two
forges, so they are never one atomic act, and the chunk reconciles what remains as the policies slice says.
