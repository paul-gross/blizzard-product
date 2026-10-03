# Carry-over

How a hub configured by file and environment becomes one configured by its store, once, with no setting left with two
owners.

## The legacy keys

| Key                                                      | Read from           |
| -------------------------------------------------------- | ------------------- |
| `[[work_source]]` blocks                                 | `blizzard-hub.toml` |
| `BZ_FORGE_URL`, `BZ_FORGE_OWNER`, `BZ_FORGE_BASE_BRANCH` | the environment     |
| `BZ_FORGE_TOKEN`, and every variable a `token_env` names | the environment     |

`hub/config.py` keeps the raw parse of `[[work_source]]` (today's `WorkSourceConfig.sources` validation) solely for the
import, and `HubConfig` no longer carries the result.

## The import

`blizzard hub config import-legacy` runs once against a migrated store, in one transaction:

1. **Secrets.** Each distinct environment variable named by a `token_env`, and `BZ_FORGE_TOKEN`, becomes a secret. The
   name is the variable's, lowercased with `_` turned to `-` (`BZ_FORGE_TOKEN` → `bz-forge-token`); a variable named by
   several sources becomes one secret they all reference.
2. **Work sources.** Each block becomes a `work_sources` row: `repo` → `locator`, `token_env` → `secret_name`, the rest
   carried as they are.
3. **Repositories.** One row per distinct `(forge, repo)` among the `git_commit` rows already in `artifacts`, each with
   `forge_api_url = BZ_FORGE_URL`, `owner` from the row's `owner/name` or `BZ_FORGE_OWNER` (falling back to the removed
   default, `blizzard`, when neither is present), `base_branch = BZ_FORGE_BASE_BRANCH` or `main`, and
   `secret_name = bz-forge-token`. These are the repositories the hub has already delivered to; a repository the hub has
   never seen is declared like any new one.
4. **Facts.** Every row written appends a `config_changes` row with `door = migration` and `actor = migration`, and one
   `config_import` fact records when the import ran and what it read.

The import is idempotent: run again after it has recorded `config_import`, it writes nothing and says so. It requires
the hub key ([secrets.md](./secrets.md) §The hub key) to exist, since it writes secrets.

## Starting with legacy keys present

The hub checks at start, after its store-revision check and in the same shape — a refusal that names the exact command
or key (`bzh:manual-migrations`):

| `config_import` recorded | Legacy keys present | Start                                                         |
| ------------------------ | ------------------- | ------------------------------------------------------------- |
| no                       | yes                 | refused, naming `blizzard hub config import-legacy`           |
| yes                      | yes                 | refused, naming each key still present and where it was found |
| either                   | no                  | starts                                                        |

A hub that never had legacy keys — a fresh one — starts with an empty store, and its first configuration arrives through
the API, the CLI, or an apply.

## The hosted hub

The hosted hub's configuration lives in its infrastructure repository: `deploy/blizzard-hub.toml` carries its
`[[work_source]]` blocks, and its secrets channel writes `BZ_FORGE_TOKEN` into `hub.env`. It does not use
`import-legacy`. Its path is config-as-code from the start:

1. The infrastructure repository gains a declarative document ([api.md](./api.md) §Apply) stating the hosted hub's work
   sources and repositories, and its secrets channel gains `BZ_HUB_SECRET_KEY`.
2. In the same change, `[[work_source]]` leaves `deploy/blizzard-hub.toml` and the `BZ_FORGE_*` keys leave `hub.env`.
3. The updater, which already runs `migrate` before swapping the hub, then sets each secret with
   `blizzard hub secret set` from the secrets channel and runs `blizzard hub config apply` with the document.

The hub version that carries this epic and that infrastructure change deploy together, so the hosted hub never starts
with legacy keys present.
