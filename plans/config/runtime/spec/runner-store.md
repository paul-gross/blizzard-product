# The runner's settings store

Where a runner host keeps its entries and its records, how a write is committed, how the store is seeded from the file,
and how a pass reads it. The routes and verbs that write it are [runner-api.md](./runner-api.md); the records it holds
for runners, workspaces, and subscriptions are shaped in [host-records.md](./host-records.md).

## Tables

Every table is added by one additive revision in `runner/store/migrations/versions/`, declared in
`runner/store/schema.py` with its `scope`, and, for a host table, its `transfer` (`bzh:runner-scoped-store`,
`bzh:host-stores-confined`). Time is stamped from the injected clock, never a `server_default` (`bzh:injected-clock`).
`contracts/store/runner.json` is regenerated, and the revision's test sits beside it (`bzh:colocated-migration-tests`).

| Table                     | Scope  | Transfer    | Holds                                                                                                                                          |
| ------------------------- | ------ | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `runner_settings`         | runner | —           | One row per runner: `runner_id` (PK), `revision` (integer, `1` at the seed), `updated_at`                                                      |
| `runner_setting_entries`  | runner | —           | `runner_id`, `key` (PK together), `value` (JSON, nullable), `at_default` (boolean), `revision` — the runner revision that last wrote it        |
| `runner_setting_changes`  | runner | —           | Append-only: `id`, `runner_id`, `recorded_at`, `revision`, `op`, `door`, `actor`, `apply_id` (nullable), `diff` (JSON `[{key, old, new}]`)     |
| `host_settings`           | host   | `none`      | One row: `revision`, `updated_at`                                                                                                              |
| `host_setting_entries`    | host   | `none`      | `key` (PK), `value`, `at_default`, `revision`                                                                                                  |
| `hosted_runners`          | runner | —           | `runner_id` (PK), `added_at`, `added_by`; the runner is the host's while its newest `hosted_runner_facts` row is not `retired`                 |
| `hosted_runner_facts`     | runner | —           | Append-only: `runner_id`, `recorded_at`, `op` (`added`, `retired`), `actor`, `door`                                                            |
| `host_workspaces`         | host   | `workspace` | `workspace_id` (PK), `revision`, `settings` (JSON — every `workspace[]` key but `id`), `created_at`                                            |
| `host_workspace_facts`    | host   | `workspace` | Append-only lifecycle: `workspace_id`, `recorded_at`, `op` (`retired`, `enabled`), `actor`, `door`                                             |
| `host_subscriptions`      | host   | `none`      | `slug` (PK), `provider`, `revision`, `settings` (JSON — `name`, `credentials_path`, `usage_api_base`, `sample_interval_seconds`), `created_at` |
| `host_subscription_facts` | host   | `none`      | Append-only lifecycle, as for workspaces                                                                                                       |
| `host_setting_changes`    | host   | `none`      | Append-only: as `runner_setting_changes`, plus `record_kind` (`host`, `workspace`, `subscription`) and `record_key`                            |

A runner's tables are runner-scoped so that `blizzard runner` transfer and ingest carry a runner's settings with it to
another host: what a runner is configured with is the runner's, not the machine's. The host's own settings stay with the
host. A workspace's record travels as the existing `workspace_prompt` row does, with the workspace.

`lease_spawns` (`runner/store/schema.py:189`) gains `host_settings_revision` and `runner_settings_revision`, both
nullable integers, written with each spawn ([passes.md](./passes.md) §Sessions). A spawn recorded before the revision
reads `null`. The revisions stay on the runner: no fact the runner sends the hub carries them.

## A write

Every write — an API write, a seed, an apply, an add or retire — is one transaction that:

1. Checks the caller's `If-Match` against the record's `revision`, refusing `409 revision_conflict` on a mismatch. An
   apply and a seed take the revision they read inside the same transaction.
2. Resolves the record's whole configuration as it would stand after the write — the file's `boot` keys, every entry,
   the change — and validates it through `parse()` (`runner/host_config.py:844`), the same validation a boot applies to
   the file. Any error refuses the whole write with each failing key named; nothing is committed.
3. Writes the changed entries, increments the record's `revision` once, and appends one change row naming every key the
   write moved, its door and its actor. A write that moves nothing increments nothing and appends nothing.

`at_default` is how a key is returned to its default without the file seeding it again at the next start: an unset
writes an entry with `at_default = true` and `value = null`, so the store holds an entry and the file's value stays
ignored (§Seeding). Reading the entry resolves to the registry default as it stands in the running build, so a default
that moves in a release reaches it.

`door` is one of `seed`, `cli`, `page`, `api`, `apply`, set as the hub sets it: by the server from the route, a client
claiming only `cli` or `page`. `actor` is the session's user, `operator` over the unix socket, and `seed` for a seed.
`op` is one of `seed`, `edit`, `create`, `retire`, `enable`.

A write is one store transaction and has no window of its own. `runner add` crosses the hub, the host's `.env`, and the
store, and its window is recorded in [host-records.md](./host-records.md) §A crash in an add.

## Seeding

At every start of `blizzard runner host`, after `ensure_current_revision` (`runner/runtime.py:465`) and before any
runner is built, the host reconciles its file into its store one entry at a time, under the rules of
[settings.md](./settings.md) §Seeding from the file, in this order:

1. The host's settings: each host-level `pass` or `session` key the file states with no entry.
2. Each `[[workspace]]` with no `host_workspaces` row, created whole from its table. Where the deprecated
   `workspace_prompt` table holds an override for that workspace, the override is the seeded `prompt`, since it already
   won over the file.
3. Each `[[subscription]]` with no `host_subscriptions` row.
4. Each `[[runner]]` whose `id` has no `hosted_runners` row: the row, an `added` fact with door `seed`, and its entries
   from the table. A `[[runner]]` whose runner has a `retired` fact is left alone: retirement is final, and the file
   cannot bring a runner back.
5. For each hosted runner, each key its table states with no entry.

Each record seeded in a step is one write of its own (§A write), so a refused entry names its table and key and fails
the start with the same message the file would fail a boot with today. A record the store already holds is never touched
by the seed, and nothing in the file is ever reported as differing from the store.

`BZ_RUNNER_TICK_SECONDS`, until its removal, seeds `tick_seconds` in step 1 only when the file states none and the store
holds none.

## The reader

`RunnerSettingsReader`, in `runner/settings/`, is the one way a pass reads entries. It answers:

- `revisions(runner_id)` — the host revision, the runner's revision, and the revision of every active workspace and
  subscription record, in one indexed read.
- `resolve(runner_id, revisions)` — the runner's `RunnerConfig` as it stands at those revisions: the file's `boot` keys,
  held from boot, overlaid with the entries. The result is cached under the revisions, so a pass whose revisions did not
  move pays one read.

`HostConfig.load` (`runner/host_config.py:567`) keeps loading the file at boot, and from then on supplies only the
`boot` keys and the seed. `RunnerConfig` keeps its shape: every consumer that reads one today reads one resolved by the
reader instead, so no consumer learns where a value came from.

The `runners[0].config` the app reads its federation, role policy, and public origins from (`runner/cli/runtime.py:995`,
`runner/app.py:463`) is replaced by the host's own `boot` keys, which those settings already are: a host whose first
runner is retired at runtime keeps serving.
