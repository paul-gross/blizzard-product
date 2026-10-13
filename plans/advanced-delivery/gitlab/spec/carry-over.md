# Carry-over

## Shipping

The slice adds no table and no column, so it carries no migration: a GitLab forge is a forge record of a new kind, and a
GitLab repository a repository record whose owner may hold a path. An installation that predates the slice lands as it
did, since every forge it has is GitHub and no GitHub origin keys differently ([namespaces.md](./namespaces.md)).

`repo_identity` and `pointer_key` change on the runner as well as at the hub, so the slice ships the runner and hub in
one release. A GitHub origin keys as before on either side, so a runner a release behind its hub keeps working for every
GitHub repository; only a GitLab repository in a nested group needs both sides at this release.

## The mock

`blizzard-mock`'s forge gains a GitLab face: `/api/v4` routes for every operation in [adapter.md](./adapter.md), over
the same git repositories its GitHub face serves, so one mock instance hosts both kinds and a chunk can span them. Its
levers cover what the proofs need — `need_rebase`, a rebase that conflicts, a failing pipeline, a merged-results
pipeline that runs on a merge ref, a retried job, a merge request passing through `locked`, a server reporting a version
below the minimum — and its answers carry GitLab's `RateLimit-*` headers, with the lever the policies slice's GitHub
face already has for setting the remaining and the reset.

## The contract suite

The mock is written from GitLab's documentation, and the behaviours above are the ones it would get wrong silently. A
contract suite therefore runs the GitLab adapter against a real GitLab CE container, pinned at the supported minimum,
covering every operation, the `locked` state, a retried job, a merged-results pipeline, and a rebase. Booting GitLab
takes minutes and needs docker, so the suite is a local-only `mise run gitlab-contract`, as the image smoke is
(`blizzard-context:/verification/blizzard/commands/packaging.md`), declared as its own method in blizzard's
verifiability matrix and owed by any change to the GitLab adapter.

## Proof

Each proof runs against the mock forge's GitLab face, and the GitLab scenarios also run in the contract suite.

- **Unchanged by default.** A fixture store captured before the slice carries over, and every GitHub chunk lands with
  the outcomes and markers it would have before.
- **A GitLab forge.** A GitLab forge is declared, a repository on it created, and `fast-forward` refused on it naming
  `refs.advance`; a server below 15.6 is refused `forge_version_unsupported` at its first call and the visit escalated
  `forge-config`.
- **The policies.** A chunk lands into a GitLab repository by `merge-on-checks` and by `park`; a change passing through
  `locked` lands rather than failing; a retried job that passed reads green; a merged-results pipeline is read and
  waited on; a person's merge of a parked merge request is found within two minutes.
- **The head gate.** A rebase the hub requested passes the gate; one that changed the set of paths, one nobody
  requested, and a person's push after the rebase do not. The crash point between the marker and the request is reached
  by the kill-9 sweep, and the next pass requests the rebase again.
- **Nested groups.** Two projects named alike in two groups on one server are two repositories, each landing its own
  chunk, and a GitHub origin's pointer key is the one it had before the slice.
- **Across kinds.** A chunk spanning a GitHub repository and a GitLab one lands both in its project's order; with the
  GitLab repository parked, the GitHub one ranked after it waits for the person's merge.
- **Budget.** With the GitLab face's rate lever lowering the remaining, a GitLab key slows rather than fails, and
  `forge-rate-limited` is recorded at most once an hour for it.
- **Invariants.** Every invariant the earlier slices declared passes after every scenario above.
