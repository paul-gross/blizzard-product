# Capabilities, narrowed to the repository

## What a repository's forge has in force

Until this epic a repository's capabilities are its forge kind's. Some are not: whether GitHub runs a merge queue on a
branch, or GitLab merge trains on a project, is set on the forge for one repository and not its neighbour, and a base
branch that only pull requests may change cannot be advanced directly however capable the kind. So a capability is now
one of two sorts, declared beside the closed capability set in `hub/domain/config/capabilities.py`:

- **Kind capabilities**, offered by every repository on a forge of that kind, as before.
- **Probed capabilities**, which a kind may offer but each repository must be shown to have: `queue.join`, and
  `refs.advance`.

`ForgeRegistry.probe(repository) -> Probes` reads the repository's forge-side settings for its base branch. A probe
belongs to the registry, beside `offered(forge, repository)`, not to any role a caller is handed: each kind's adapter
supplies the reads below, and the registry calls them.

| Capability     | GitHub                                                                                                                                                                                                                                                           | GitLab                                                              |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `queue.join`   | GraphQL `repository.mergeQueue(branch: <base>)` answers a queue, which finds one whether a ruleset or classic protection requires it                                                                                                                             | `merge_trains_enabled` and `merge_pipelines_enabled` on the project |
| `refs.advance` | No `pull_request`, `merge_queue`, `required_linear_history`, `required_signatures`, or `update` rule among the base's active rules (`GET /repos/{o}/{r}/rules/branches/{base}`), and no advance of this base refused as protected since the last on-demand probe | Never offered by the kind                                           |

A merge queue in force on the base also refuses a direct merge of a change, so a probe that shows `queue.join` leaves
`changes.merge.merge`, `changes.merge.squash`, and `changes.merge.rebase` out of what the repository is offered, and a
`merge-on-checks` repository whose operator turns a queue on is escalated to a person at its next visit
(`forge-capability-missing`, naming the capability and pointing at `forge-queue`), rather than failing every merge.

Classic branch protection is unreadable in detail without admin access, though whether a branch is protected at all is
not (`protected` on `GET /repos/{o}/{r}/branches/{base}`), and a probe records it. An advance the forge refuses as
protected is remembered on the probe as `advance_refused_at`, and leaves `refs.advance` out from then on, however many
scheduled probes follow, until an operator takes one on demand — the act that says the base's protection has changed. A
train refused once therefore never loops into the same refusal.

`ForgeRegistry.offered(forge, repository)` — the one place a caller asks, as the forges epic made it — answers the
kind's capabilities, narrowed by the repository's last probe. A probe is taken when a repository is created, when its
policy, its batching, or its forge changes, and whenever its cached answer is older than ten minutes at the moment a
visit's capability check asks. A probe the forge cannot answer — a 5xx, a timeout, a rate limit — changes nothing: the
last answer stands and is retried at the next check. A repository never yet probed has no answer to stand on, so a visit
whose capability check needs a probed capability answers `pending` until a probe succeeds, rather than refusing work
because the forge blinked. `GET /api/repositories/{repository}` carries the last probe, when it was taken, and its
`advance_refused_at`, and `blizzard hub repo probe <repository>` takes one on demand and clears `advance_refused_at`.

## New seam operations

The flow's own train builds and cleans up its branches with `create_ref` and `delete_ref`, which the `policies` epic
adds to `IForgeRefs` for fast-forward's landing branch. It needs one more write, joining the role its kind of work
belongs to:

| Operation                       | Role            | Capability      | GitHub                                         |
| ------------------------------- | --------------- | --------------- | ---------------------------------------------- |
| `close_change(number, comment)` | `IForgeChanges` | `changes.close` | `PATCH /repos/{o}/{r}/pulls/{n}` and a comment |

It is async, as every role's operation is, and answers through the forges epic's retry policy. GitHub offers it. GitLab
offers `changes.close` alone of the writes a train needs; with neither `branches.merge` nor `refs.advance` it cannot
host the flow's own train, and its repositories hand their line to GitLab's merge train instead
([queue.md](./queue.md)). The queue's own role is `queue.md`'s.

## The batching settings

`repositories` gains how its flow batches, as configured fields under the record's verbs and revisions, beside the
policy the policies epic added:

```text
repositories
  batch_size            Integer  NOT NULL   1 to 20; 1 lands each submission alone, as the flows epic does
  batch_wait_seconds    Integer  NOT NULL   how long the oldest ready submission waits for others; 60 by default
  train_prefix          String   NOT NULL   the branch prefix trains are proven under; blizzard-train/ by default
  staging_prefix        String   NOT NULL   the branch prefix trains are assembled under; blizzard-staging/ by default
```

`RepositoryFields`, `RepositoryEdit`, `_FIELDS`, and `_DIFF_FIELDS` gain them. A `batch_size` above one is legal only
for a repository whose policy the hub lands itself — `fast-forward` or `merge-on-checks` — and is refused
`422 batching_not_applicable` beside `park` or `forge-queue`. It requires `refs.create`, `refs.delete`,
`branches.merge`, `refs.advance`, and `checks.read`, and `changes.close` for `merge-on-checks`, checked against
`offered(forge, repository)` when it is set, refused `422 forge_capability_missing` as a policy is, and again as each
batch forms: a flow whose repository has lost one since forms no batch, and its ready members fall back to landing
alone, with a `forge-capability-fallback` event naming the capability. Because a probe leaves `refs.advance` out
wherever the base requires linear history or signed commits, a repository with either cannot batch: a train lands merge
commits, whatever squash or rebase convention the repository's people keep. `train_prefix` and `staging_prefix` must be
valid branch prefixes, may not be the base branch or a prefix of it, and may not be prefixes of each other.

A repository that batches must run its checks on pushes to its `train_prefix` branches, since a train is a branch the
hub pushes and nothing else would run them, and should not run them on its `staging_prefix` branches, where a train is
assembled one merge at a time. Which checks run there is the repository's choice: the train reads them with the reading
every policy uses, and a repository that runs only a linter on its train branches is proven by its linter. The admin
form says so beside the fields, and a train that sees no check is abandoned rather than waited on forever
([batches.md](./batches.md#proving)).
