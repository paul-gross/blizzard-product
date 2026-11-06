# The runner's settings store

Where a runner host keeps its entries and its records, how a write is committed, how the store is seeded from the file,
and how a pass reads it. The routes and verbs that write it are [runner-api.md](./runner-api.md); the records it holds
for runners, workspaces, and subscriptions are shaped in [host-records.md](./host-records.md).

## Tables

Every table is added by one additive revision in `runner/store/migrations/versions/`, declared in
`runner/store/schema.py` with its `scope`, and, for a host table, its `transfer` (`bzh:runner-scoped-store`,
`bzh:host-stores-confined`). Time is stamped from the injected clock, never a `server_default` (`bzh:injected-clock`).
`contracts/store/runner.json` is regenerated, and the revision's test sits beside it (`bzh:colocated-migration-tests`).

| Table                     | Scope  | Transfer                                                                      | Holds                                                                                                                                          |
| ------------------------- | ------ | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `runner_settings`         | runner | —                                                                             | One row per runner: `runner_id` (PK), `revision` (integer, `1` at the seed), `updated_at`                                                      |
| `runner_setting_entries`  | runner | —                                                                             | `runner_id`, `key` (PK together), `value` (JSON, nullable), `at_default` (boolean), `revision` — the runner revision that last wrote it        |
| `runner_setting_changes`  | runner | —                                                                             | Append-only: `id`, `runner_id`, `recorded_at`, `revision`, `op`, `door`, `actor`, `apply_id` (nullable), `diff` (JSON `[{key, old, new}]`)     |
| `host_settings`           | host   | `none`                                                                        | One row: `revision`, `updated_at`                                                                                                              |
| `host_setting_entries`    | host   | `none`                                                                        | `key` (PK), `value`, `at_default`, `revision`                                                                                                  |
| `hosted_runners`          | host   | `none`; ingest writes the destination's row                                   | `runner_id` (PK), `added_at`, `added_by`; the runner is the host's while its newest `hosted_runner_facts` row is not `retired`                 |
| `hosted_runner_facts`     | host   | `none`; ingest writes the destination's `added` fact                          | Append-only: `runner_id`, `recorded_at`, `op` (`added`, `retired`), `actor`, `door`                                                            |
| `host_workspaces`         | host   | `none`                                                                        | `workspace_id` (PK), `revision`, `settings` (JSON — every `workspace[]` key but `id`), `created_at`                                            |
| `host_workspace_facts`    | host   | `none`                                                                        | Append-only lifecycle: `workspace_id`, `recorded_at`, `op` (`retired`, `enabled`), `actor`, `door`                                             |
| `host_subscriptions`      | host   | `none`                                                                        | `slug` (PK), `provider`, `revision`, `settings` (JSON — `name`, `credentials_path`, `usage_api_base`, `sample_interval_seconds`), `created_at` |
| `host_subscription_facts` | host   | `none`                                                                        | Append-only lifecycle, as for workspaces                                                                                                       |
| `runner_hub_tokens`       | host   | `none`; ingest re-seals the moving runner's token under the destination's key | `runner_id` (PK), `ciphertext`, `key_id`, `revision`, `updated_at` — the runner's hub token, sealed (§Sealed tokens)                           |
| `host_setting_changes`    | host   | `none`                                                                        | Append-only: as `runner_setting_changes`, plus `record_kind` (`host`, `workspace`, `subscription`) and `record_key`                            |

A runner's settings tables are runner-scoped (`bzh:runner-scoped-store`) so that `blizzard runner` transfer and ingest
carry a runner's settings with it to another host: what a runner is configured with is the runner's, not the machine's.
Each is read and written only through that runner's bound connection, never a connection handed a `runner_id`. Which
runners a host holds is the host's fact, so `hosted_runners` and its facts are host tables, read through `HostStores`
(`bzh:host-stores-confined`); the host tick lists its runners there and reads each one's settings through the bound
connection it already builds for that runner. The host's own settings, workspaces, subscriptions, and tokens stay with
the host: ingest carries none of them, and the destination's records stand.

An add and a seed write the runner's `runner_identity` row in the same transaction as its settings, since the id is
known — the hub returned it, or the file states it — so no settings row ever names a runner the store does not know, and
`runner:rows-name-registered-runner` (`runner/store/internal/invariants.py:193`) holds after every crash point. A
retirement keeps the identity row with the settings rows it describes. `architecture/system-shape/runner-scoping.md`
§Scope, which says a runner's binding is made once from the host file's `id`, and
`architecture/crash-correctness/runner.md` §Binding at the configured id, which says it is reseeded from the file,
change to say it is made from `hosted_runners`.

`lease_spawns` (`runner/store/schema.py:189`) gains `host_settings_revision` and `runner_settings_revision`, both
nullable integers, and `transcripts_root`, nullable text, written with each spawn ([passes.md](./passes.md) §Sessions).
A spawn recorded before the revision reads `null`. The revisions stay on the runner: no fact the runner sends the hub
carries them.

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
2. Each `[[workspace]]` with no `host_workspaces` row, created whole from its table. A file with no `[[workspace]]`
   seeds none: there is no default workspace (§No default workspace). Where the deprecated `workspace_prompt` table
   holds an override for that workspace, the override is the seeded `prompt` and the record's `prompt_file` and
   `prompt_package` are left at their defaults, since the override already won over every prompt the file states;
   seeding it beside a `prompt_file` or `prompt_package` therefore neither loses it nor fails the start.
3. Each `[[subscription]]` with no `host_subscriptions` row.
4. Each `[[runner]]` whose `id` has no `hosted_runners` row: the row, an `added` fact with door `seed`, its
   `runner_identity` row, and its entries from the table. A `[[runner]]` with no `id` — a scaffold that has not yet
   joined the hub — is left for its join, which seeds it. A `[[runner]]` whose runner has a `retired` fact is left
   alone: retirement is final, and the file cannot bring a runner back.
5. For each hosted runner, each key its table states with no entry.
6. For each hosted runner with no `runner_hub_tokens` row, its token as the host resolves it today — the process
   environment's `BZ_HUB_TOKEN_<NAME>`, then `[hub_tokens]`, then `.env` (`runner/host_config.py:237`) — sealed into the
   store (§Sealed tokens).

Each record seeded in a step is one write of its own (§A write), so a refused entry names its table and key and fails
the start with the same message the file would fail a boot with today. A record the store already holds is never touched
by the seed, and nothing in the file is ever reported as differing from the store.

`BZ_RUNNER_TICK_SECONDS`, until its removal, seeds `tick_seconds` in step 1 only when the file states none and the store
holds none.

## Sealed tokens

A runner's hub token moves from the host's `.env`, keyed by the runner's name, into `runner_hub_tokens`, keyed by its
id, sealed with AES-256-GCM under a host key and bound to the runner's id and the row's revision, so a ciphertext copied
to another row does not open. Nothing about a runner's token then depends on its name, and renaming a runner is one
store write.

- **The sealing is the hub's, shared.** The hub's sealing (`hub/secrets/internal/aes_gcm.py`, `key_material.py`), its
  key providers (`env_keys.py`, `directory_keys.py`), and its rotation (`hub/secrets/rotation.py`) move to
  `foundation/secrets/`, the hub keeping its own keys and behavior exactly as they are. This is pre-work, a piece of its
  own ([carry-over.md](./carry-over.md)).
- **The host's key.** `BZ_HOST_SECRET_KEY`, with `BZ_HOST_SECRET_KEY_PREVIOUS` readable during a rotation, as the hub's
  key is supplied; where neither is set, a key directory under the host's runtime root, minted at the first start that
  needs it, owner-only, declared as a durable path of the host registry. The variables supply no key of the file and
  stay outside the registry ([lifetimes.md](./lifetimes.md) §Environment variables).
- **Reading a token.** The host opens a runner's token when it builds the runner's hub clients and when the token
  changes, never at a pass; `HubTokenHolder` (`runner/composition.py:303`), which re-reads `.env` while the runner's
  identity is in doubt, re-reads the row instead.
- **Writing one.** An add writes the token the hub returned beside the runner's rows, in the add's transaction; a
  retirement deletes the ciphertext and keeps the row's history in the change log. No route, view, or change row ever
  carries a token or its ciphertext.
- **What the old places become.** `BZ_HUB_TOKEN_<NAME>`, `[hub_tokens]`, and the `.env` variables are read only by the
  seed (§Seeding step 6), deprecated, and named in the upgrade note; `runner add` stops writing `.env`, and
  `runner retire` stops editing it. `runners/<name>/`, which `add` and `ingest` create and nothing reads, stops being
  created and its durable path is deprecated. Their removal is a follow-up `(BREAK)`.

## No default workspace

A host has the workspaces it is given and no other. Today a configuration built without a `[[workspace]]` falls back to
one named `workspace-local` (`runner/config.py:83`, `:768`; `runner/host_config.py:562`), and `runner init` scaffolds a
`[[workspace]]` under that id from `BZ_WORKSPACE_ROOT` and its siblings (`runner/host_config.py:681`). Both go: a file
with no `[[workspace]]` gives a host with no workspace, `init` scaffolds none, and the `BZ_WORKSPACE_*` variables `init`
read are no longer read. A host with no workspace runs, reports none, and its runners claim nothing until the operator
creates one ([host-records.md](./host-records.md) §Workspaces and subscriptions) and binds a project to it.

This is a break, made on the owner's word without a deprecation window, since no deployed host relies on the fallback:
every host `init` scaffolded carries its `[[workspace]]` in its file, and it is seeded as any other. The change note
names it as breaking. A legacy flat runner file still graduates to a host file naming the workspace it always named.

## The reader

`RunnerSettingsReader`, in `runner/settings/`, is the one way a pass reads entries. Each runner's reader is built over
that runner's bound connection and the host's `HostStores`, so it is never handed a `runner_id`, and the host tick holds
one per hosted runner. It answers:

- `revisions()` — the host revision, the runner's revision, and the revision of every active workspace and subscription
  record, in one indexed read.
- `resolve(revisions)` — the runner's `RunnerConfig` as it stands at those revisions: the file's `boot` keys, held from
  boot, overlaid with the entries. The result is cached under the revisions, so a pass whose revisions did not move pays
  one read.

`HostConfig.load` (`runner/host_config.py:567`) keeps loading the file at boot, and from then on supplies only the
`boot` keys and the seed. `RunnerConfig` keeps its shape: every consumer that reads one today reads one resolved by the
reader instead, so no consumer learns where a value came from.

The `runners[0].config` the app reads its federation, role policy, and public origins from (`runner/cli/runtime.py:995`,
`runner/app.py:463`) is replaced by the host's own `boot` keys, which those settings already are: a host whose first
runner is retired at runtime keeps serving.
