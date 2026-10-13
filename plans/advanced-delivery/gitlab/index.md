# Plan — `epic:advanced-delivery`, gitlab slice

A shop whose repositories live on GitLab cannot use blizzard to land anything today, and after the `policies` slice it
still cannot. The seam the `forges` slice drew speaks in refs, changes, and checks rather than in GitHub's words, and
every policy is written against that seam alone, but the only forge behind it is GitHub. A seam with one occupant is a
promise rather than a proof: nothing yet shows that a policy written for one forge runs unchanged on another.

This slice puts GitLab behind the seam. A tenant declares a GitLab forge as it declares a GitHub one, a repository on it
names a policy as any repository does, and the hub lands into it through merge requests and pipelines without a policy,
a graph, or the combining step learning which forge it holds. A chunk whose work spans a GitHub repository and a GitLab
one lands into both, each through its own forge, in its project's order.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **GitLab as a forge kind.** An operator declares a GitLab server — gitlab.com or their own — as a forge, and a
  repository on it lands the way a GitHub repository does. The adapter speaks GitLab's merge requests, pipelines,
  branches, and comparisons, and maps each of GitLab's words into the seam's, so the policies, the capability check, and
  the flow never read GitLab's vocabulary. A server too old to answer what the adapter relies on is refused when the
  forge is first used, naming the version it found.
- **The policies GitLab can honour.** Merging once checks pass and parking for a person run on GitLab as they do on
  GitHub. GitLab has no way to move a branch outside a merge request, so a GitLab repository cannot fast-forward, and
  setting that policy on one is refused naming what the forge lacks. A project that merges by its own method — a merge
  commit, semi-linear history, or a fast-forward — keeps that method, and the hub records whatever revision the merge
  produced.
- **Checks read where GitLab runs them.** A project that runs merged-results pipelines, or runs its pipelines only for
  merge requests, runs its checks on a merge GitLab builds rather than on the change's head. The hub reads a change's
  own pipeline, so such a project's checks are seen and waited on rather than missed, and a job that failed and was
  retried counts once, at its latest outcome.
- **A rebase the hub asked for is still the chunk's work.** GitLab brings a change up to its base by rebasing it,
  rewriting its head, and GitLab's comparisons cannot prove a rebase carried the work unchanged the way a merge of the
  base can be proven on GitHub. The hub admits a rebased head only when it asked for that rebase itself and what GitLab
  can say about the result matches what it rebased; any other head is foreign, as it is today.
- **Repositories in nested groups.** A GitLab project may live several groups deep. A repository on GitLab names its
  whole namespace, and two projects of the same name in different groups on one server stay two repositories, on the
  runner as on the hub.
- **Within GitLab's limits.** GitLab counts requests by the minute rather than the hour, and may say how much remains
  only once it refuses. The rate budget learns from whatever GitLab tells it, and a GitLab key slows down rather than
  fails exactly as a GitHub key does.
- **Landing across kinds of forge.** A chunk whose project links a GitHub repository and a GitLab one lands into both,
  each through its own forge and with its own policy, in its project's order. Landing across forges is never one atomic
  act, and the rule the `policies` slice set for landing across repositories holds unchanged.
- **Held to GitLab's real behaviour.** The mock forge gains a GitLab face for the fleet's own proofs, but a mock written
  from documentation is wrong in exactly the ways that matter, so a contract suite runs the adapter against a real
  GitLab and is owed by any change to it.

## Left for later

GitLab's merge trains, as a way to hand a repository's line to its forge, are the `trains` slice's, which comes after
this one and speaks GitHub's merge queue and GitLab's merge train alike. GitLab as a source of work items, rather than a
place work lands, is not this epic's. A forge beyond GitHub and GitLab waits for a shop that needs one, and this slice
is the evidence that adding it means writing an adapter rather than revisiting the policies.
