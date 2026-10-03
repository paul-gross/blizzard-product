# Configured records

The work sources and repositories the hub holds as data, how each change to them is recorded, and how delivery resolves
a chunk's commits against them. Secrets are [secrets.md](./secrets.md)'s; the verbs that change a record are
[api.md](./api.md)'s.

## What moves

| Today                                                                                | Becomes                                                                                     |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `[[work_source]]` blocks, parsed by `hub/config.py::WorkSourceConfig.sources`        | `work_sources` rows                                                                         |
| `WorkSourceConfig.token_env`, a variable name the hub reads from its own process     | `work_sources.secret_name`, a reference into the secret store                               |
| `BZ_FORGE_URL`, `BZ_FORGE_OWNER`, `BZ_FORGE_BASE_BRANCH` read in `hub/app.py`        | `repositories` rows                                                                         |
| `BZ_FORGE_TOKEN` read in `hub/app.py`                                                | a secret, referenced by `repositories.secret_name`                                          |
| `hub/app.py::DEFAULT_FORGE_OWNER = "blizzard"`, `DEFAULT_FORGE_BASE_BRANCH = "main"` | removed — a repository declares its owner and base branch, and nothing falls back to a name |
| `HubConfig.work_sources`                                                             | removed; [carry-over.md](./carry-over.md) reads the legacy keys once                        |

## Work sources

`work_sources` holds one row per configured source. The built-in `hub` source has no row: `seat_hub_work_source`
(`hub/work_sources/internal/factory.py`) keeps seating it, and the API lists it as built-in, never editable or
retirable.

| Column        | Type        | Meaning                                                                                                                                          |
| ------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `name`        | String, PK  | The ingest-token prefix and the key `chunk_work_refs.source` already stores. Immutable: work refs, work items, and closures key on it.           |
| `provider`    | String      | One of `_KNOWN_WORK_SOURCE_PROVIDERS` (`hub/config.py`), checked on write.                                                                       |
| `locator`     | String      | Where the provider's items live. For `github`, today's `repo` (`owner/name`). Provider-defined so a tracker addressed by site fits the same row. |
| `api_base`    | String NULL | Provider API origin override; today's `api_base`.                                                                                                |
| `web_base`    | String NULL | Web origin override; derived from `api_base` when null, as `GithubEntry.web_base` does now.                                                      |
| `annotate`    | Boolean     | Opts into the forge-status sweep.                                                                                                                |
| `secret_name` | String NULL | FK `secrets.name`. Null only for a provider that needs no credential.                                                                            |
| `managed_by`  | String NULL | The label of the declarative document that owns the row ([api.md](./api.md) §Apply); null when created by any other door.                        |
| `revision`    | Integer     | 1 at create, incremented by every committed change to the row.                                                                                   |
| `created_at`  | UtcDateTime | `bzh:utc-instants`.                                                                                                                              |
| `created_by`  | String      | The acting user id, or `migration`.                                                                                                              |

Constraints carried over from `WorkSourceConfig.sources`, now enforced on write: `name` contains no `:` (the
ingest-token grammar splits on the first colon), `name` is not `hub` (`RESERVED_HUB_SOURCE_NAME`), and
`UniqueConstraint("provider", "locator")` — two names for one locator would let the same item be ingested twice under
two identities.

`work_source_lifecycle_facts` holds retirement in the shape `scope_lifecycle_facts` already has — `id`, `name`,
`retired`, `set_at`, `set_by` — append-only, the newest row deciding (`bzh:facts-not-status`).

## Repositories

`repositories` holds one row per repository work lands in.

| Column          | Type        | Meaning                                                                           |
| --------------- | ----------- | --------------------------------------------------------------------------------- |
| `name`          | String, PK  | The handle a bare commit repo resolves against; immutable.                        |
| `forge_api_url` | String      | Today's `BZ_FORGE_URL`.                                                           |
| `owner`         | String      | The forge owner; with `repo`, the `owner/name` coordinate every forge route uses. |
| `repo`          | String      | The repository's name on the forge.                                               |
| `base_branch`   | String      | Today's `BZ_FORGE_BASE_BRANCH`.                                                   |
| `secret_name`   | String      | FK `secrets.name`; today's `BZ_FORGE_TOKEN`.                                      |
| `managed_by`    | String NULL | As on `work_sources`.                                                             |
| `revision`      | Integer     | As on `work_sources`.                                                             |
| `created_at`    | UtcDateTime |                                                                                   |
| `created_by`    | String      |                                                                                   |

`UniqueConstraint("forge_api_url", "owner", "repo")`. `repository_lifecycle_facts` mirrors
`work_source_lifecycle_facts`.

### Resolving a chunk's commits

A `git_commit` row in `artifacts` carries `repo` (`owner/name`, or a bare name when the origin named no owner) and
`forge` (the origin URL). Delivery resolves each of a chunk's latest commit pointers to one repository row:

1. `forge`'s host matches `forge_api_url`'s forge and `repo` equals `owner/repo` — the exact match.
2. Otherwise a bare `repo` equal to a row's `name`, or to a row's `repo` when exactly one row carries it.
3. Otherwise the commit is unresolved.

The `run:` step environment keeps every variable `bzh:hub-node-env-contract` (`standards/hub-nodes/env-contract.md`)
lists, under the same names; only their source changes. `BZ_FORGE_URL`, `BZ_FORGE_OWNER`, `BZ_HUB_BASE_BRANCH`, and
`BZ_FORGE_TOKEN` are filled from the resolved rows, and each `BZ_HUB_GIT_COMMITS` entry's `repo` is qualified to the
row's `owner/repo`, so `LandRun.repo` (`hub/graphs/scripts/land_common.py`) never depends on an owner fallback. Those
variables are single-valued, so the chunk's resolved rows must agree on `forge_api_url`, `owner`, `base_branch`, and
`secret_name`. An unresolved commit, or rows that disagree, refuses the hub step before its command runs, with a named
outcome — `repository-unresolved` or `repositories-disagree` — recorded in `event_log` and routed as a failed hub step.
Landing across forges is `epic:advanced-delivery`'s. `GitHubCommitResolver` (`hub/forge/internal/commit_resolver.py`)
takes the forge URL, owner, and token per call from the resolved row instead of from its constructor.

## The change log

Every committed write to a configured record appends one `config_changes` row in the same transaction as the write.

| Column        | Type        | Meaning                                                                                                    |
| ------------- | ----------- | ---------------------------------------------------------------------------------------------------------- |
| `id`          | Integer, PK | Durable write order.                                                                                       |
| `recorded_at` | UtcDateTime |                                                                                                            |
| `actor`       | String      | The acting user id, or `migration`.                                                                        |
| `door`        | String      | `board`, `cli`, `api`, `apply`, or `migration`.                                                            |
| `record_kind` | String      | `work_source`, `repository`, `secret`, `scope`, `routine`.                                                 |
| `record_key`  | String      | The record's key.                                                                                          |
| `revision`    | Integer     | The record's revision after this change.                                                                   |
| `op`          | String      | `create`, `edit`, `retire`, `enable`, or `replace` (a secret's value).                                     |
| `diff`        | Text (JSON) | `[{field, old, new}]` for the fields that changed. A secret's value never appears; `replace` carries none. |
| `apply_id`    | String NULL | Groups the rows one apply wrote.                                                                           |

`door` comes from the `X-Blizzard-Door` request header the CLI and board send, defaulting to `api`; an apply sets
`apply` and the carry-over `migration`. Reads of the log are newest-first and page-bounded (`bzh:page-bounded-read`),
with an explicit total order on `id` (`bzh:sql-portable`). The change log is fact rows; the record row is a mutable
entity whose `revision` changes in place with its fields, the same recorded-state position `work_items` holds under
`bzh:facts-not-status`.

## Uniqueness

Names are unique across the hub. `epic:multi-tenancy` adds `tenant_id` to each table here and makes every uniqueness
rule above `(tenant_id, …)`. Work sources and repositories stay tenant-wide records: `epic:projects` lets a project link
the ones it draws from and lands in, and a chunk then lands only in repositories its project links. Nothing here assumes
a single source or a single repository.

## Seams

Each record kind has a read and a write repository Protocol (`bzh:repository-split`) with a plural reconstitution
(`bzh:bulk-reconstitution`), patterned on `exemplars/python/repo_pattern.py`. API and CLI controllers hold the read
halves only (`bzh:controller-read-only`). Writes go through one domain service, `ConfigAuthoring`, which takes loaded
records (`bzh:domain-takes-objects`) and writes the row, its lifecycle fact where one applies, and its `config_changes`
row in one transaction. The tables arrive by one Alembic revision applied through `blizzard hub migrate`
(`bzh:manual-migrations`), inside the portable surface (`bzh:sql-portable`).
