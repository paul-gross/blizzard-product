# Capabilities, narrowed to the repository

## What a repository's forge has switched on

Until this slice a repository's capabilities are its forge kind's. Some are not: whether GitHub runs a merge queue on a
branch, or GitLab merge trains on a project, is set on the forge for one repository and not its neighbour, and a base
branch that only pull requests may change cannot be advanced directly however capable the kind. So a capability is now
one of two sorts, declared beside the closed set in `hub/forge/landing.py`:

- **Kind capabilities**, offered by every repository on a forge of that kind, as before.
- **Probed capabilities**, which a kind may offer but each repository must be shown to have: `queue.join`, and
  `refs.advance`.

`IForgeLanding` gains `probe(base) -> Probes`, which reads the repository's forge-side settings for its base branch:

| Capability     | GitHub                                                                                                                                                   | GitLab                                                              |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `queue.join`   | A `merge_queue` rule among the base's active rules (`GET /repos/{o}/{r}/rules/branches/{base}`)                                                          | `merge_trains_enabled` and `merge_pipelines_enabled` on the project |
| `refs.advance` | No `pull_request` or `merge_queue` rule among the base's active rules; classic protection, unreadable without admin, is found when an advance is refused | Never offered by the kind                                           |

`ForgeRegistry.offered(repository)` — the one place a caller asks, as the forges slice made it — answers the kind's
capabilities, less each probed one the repository's last probe did not show. A probe is taken when a repository is
created, when its policy, its batching, or its forge changes, and whenever its cached answer is older than ten minutes
at the moment a visit's capability check asks. A probe the forge cannot answer leaves the probed capabilities out, so
the hub assumes the narrower forge until it is shown the wider one. `GET /api/repositories/{repository}` carries the
last probe and when it was taken, and `blizzard hub repo probe <repository>` takes one on demand.

## New seam operations

The flow's own train needs three more writes, each behind a kind capability:

| Operation                       | Capability      | GitHub                                         |
| ------------------------------- | --------------- | ---------------------------------------------- |
| `create_ref(branch, revision)`  | `refs.create`   | `POST /repos/{o}/{r}/git/refs`                 |
| `delete_ref(branch)`            | `refs.delete`   | `DELETE /repos/{o}/{r}/git/refs/heads/{b}`     |
| `close_change(number, comment)` | `changes.close` | `PATCH /repos/{o}/{r}/pulls/{n}` and a comment |

GitHub offers all three. GitLab offers `changes.close` alone; with neither `branches.merge` nor `refs.advance` it cannot
host the flow's own train, and its repositories hand their line to GitLab's merge train instead
([queue.md](./queue.md)). The queue's own operations are `queue.md`'s.

## The batching settings

`repositories` gains how its flow batches, as configured fields under the record's verbs and revisions, beside the
policy the policies slice added:

```text
repositories
  batch_size            Integer  NOT NULL   1 to 20; 1 lands each submission alone, as the flows slice does
  batch_wait_seconds    Integer  NOT NULL   how long the oldest ready submission waits for others; 60 by default
  train_prefix          String   NOT NULL   the branch prefix trains are built under; blizzard-train/ by default
```

`RepositoryFields`, `RepositoryEdit`, `_FIELDS`, and `_DIFF_FIELDS` gain them. A `batch_size` above one is legal only
for a repository whose policy the hub lands itself — `fast-forward` or `merge-on-checks` — and is refused
`422 batching_not_applicable` beside `park` or `forge-queue`. It requires `refs.create`, `refs.delete`,
`branches.merge`, `refs.advance`, and `checks.read`, and `changes.close` for `merge-on-checks`, checked against
`offered(repository)` when it is set and again at each batch, refused `422 forge_capability_missing` as a policy is.
`train_prefix` must be a valid branch prefix and may not be the base branch or a prefix of it.

A repository that batches must run its checks on pushes to its `train_prefix` branches, since a train is a branch the
hub pushes and nothing else would run them. The admin form says so beside the field, and a train that sees no check is
abandoned rather than waited on forever ([batches.md](./batches.md#proving)).
