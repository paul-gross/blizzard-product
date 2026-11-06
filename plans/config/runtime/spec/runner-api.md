# The runner's settings API

The routes a runner host serves its settings through, who may write them, the document an apply takes, the CLI verbs
over the routes, and the local page. Every route lives in `runner/api/`, is described in `openapi/runner.openapi.json`,
and reaches the local page through its generated client (`bzh:generated-client`). Every route is additive to the
runner's API (`bzh:released-surface-additive`).

## Who may write

Reads sit on the host's human lane, open to anyone the local page admits. Every write — a settings edit, an apply, a
runner's add or retire, a workspace's or subscription's create, edit, retire, or enable — requires the `superuser` role
of the session (`runner/auth/roles.py`), checked by a `require_superuser` dependency in `runner/api/wiring.py`, and is
refused `403 superuser_required` otherwise. It is the first route on the runner that checks a role: today every
signed-in human may call every human-lane write. A caller over the unix socket holds the implicit superuser session
(`runner/auth/session.py:114`), so the CLI writes as it reaches the host today, its permission the socket file's. A
worker reaches only the ungated lane under its lease token and can reach none of these routes.

The workspace prompt's existing routes write a workspace's record ([host-records.md](./host-records.md) §Workspaces and
subscriptions) and take the superuser check with every other settings write. The other existing human-lane writes —
pause, takeover, requeue, selftest — keep the access they have; narrowing them is not this epic's.

## Routes

`{key}` in a body or a query is a key's dotted path as the contract document spells it. A table key's whole value is one
entry and is set whole.

| Route                                                                        | Does                                                                                                    |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `GET /api/host/settings`                                                     | The host-level settings view (§Views)                                                                   |
| `PATCH /api/host/settings`                                                   | A settings write over the host-level entries (§A settings write)                                        |
| `GET /api/host/settings/changes`                                             | The host's change log, newest first, paginated; `?record_kind=` and `?record_key=` narrow it            |
| `GET /runners/{runner_id}/api/settings`                                      | The runner's settings view                                                                              |
| `PATCH /runners/{runner_id}/api/settings`                                    | A settings write over the runner's entries                                                              |
| `GET /runners/{runner_id}/api/settings/changes`                              | The runner's change log                                                                                 |
| `POST /api/host/settings/apply`                                              | A declarative apply (§Apply); `?dry_run=true` rolls it back                                             |
| `GET /api/host/settings/export`                                              | The document an apply takes, as the store stands: every entry, every active record, every hosted runner |
| `GET`, `POST /api/host/runners`; `POST /api/host/runners/{runner_id}/retire` | The hosted runners ([host-records.md](./host-records.md) §Runners)                                      |
| `/api/host/workspaces`, `/api/host/subscriptions`                            | The six verbs of a record ([host-records.md](./host-records.md) §Workspaces and subscriptions)          |

### Views

A settings view answers what the runner is configured with and what it is running with:

```json
{
  "revision": 7,
  "in_force": { "revision": 6, "since": "2026-11-05T22:14:03Z" },
  "settings": [
    { "key": "throttle.budget.usd", "value": 40.0, "source": "entry", "lifetime": "pass",
      "revision": 7, "written_at": "2026-11-05T22:15:40Z", "written_by": "operator" },
    { "key": "agents", "value": 2, "source": "default", "lifetime": "pass" }
  ]
}
```

- `revision` is the record's revision in the store; `in_force` is the revision the runner's most recent pass started
  from, and when it took hold. The two differ for the length of one tick after a write, and the operator reads the gap
  as "written, not yet running".
- `source` is `entry`, or `default` for a key with no entry or an entry `at_default`. Every `pass`, `session`, and
  `immutable` key the registry declares for the record is listed, so the view is the whole answer to what can be
  changed. A `boot` key is never listed: it is the daemon's footing, it holds the host's credentials and its store's
  address, and it is changed in the file, not here. No `pass`, `session`, or `immutable` key holds a secret — a
  subscription's `credentials_path` and OpenCode's `auth_path` are paths to credentials, never the credentials — and a
  key added later that would hold one is a question for its own plan, not a row in this view.
- The host view lists the host-level keys and its own `in_force`, taken from the host tick.

### A settings write

```json
{ "set": { "throttle.budget.usd": 40.0, "queue.strict": true }, "unset": ["agents"] }
```

sent with `If-Match: <revision>`. `set` replaces each named entry whole; `unset` returns each named key to its default
(`at_default`, [runner-store.md](./runner-store.md) §A write); a key named in neither is untouched. A key in both is
refused. It answers the new view, or:

| Refusal                  | When                                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------------- |
| `409 revision_conflict`  | `If-Match` is missing or stale                                                                          |
| `422 setting_unknown`    | A key the registry does not declare for the record                                                      |
| `422 setting_not_live`   | A `boot` key — its message names `blizzard-host.toml` and a restart                                     |
| `422 setting_immutable`  | An `immutable` key                                                                                      |
| `422 settings_invalid`   | The resolved configuration fails validation; `errors[]` names each key and its message, as a boot would |
| `403 superuser_required` | The session is not the superuser                                                                        |

A write against a runner whose newest hosted fact is `retired` is refused `409 runner_retired`.

## Apply

An apply takes a document shaped like `blizzard-host.toml` — the same tables, the same key names — and reconciles the
store to what it states, under the rules `bzh:config-apply` gives the hub's document:

- **A door, not an owner.** A stated host-level key or runner key is set; a stated `[[workspace]]` or `[[subscription]]`
  with no record is created, one with a record is edited by exactly the keys it states, and one retired is enabled. A
  key the document does not state is untouched, and a record it does not name is never retired.
- **Runners by id.** A `[[runner]]` is matched by `id`. One whose `id` the host does not hold is refused
  `runner_unknown`: an apply sets a runner's settings and never adds one, since adding one is minted at the hub
  ([host-records.md](./host-records.md) §Runners).
- **`boot` keys are refused** with `setting_not_live`, `immutable` keys with `setting_immutable` unless they equal the
  record's, which is how a document names the record it means.
- **One transaction.** Every write of the apply commits together, each beside its change row, door `apply`, sharing one
  `apply_id`, or none does. A dry run is the same transaction rolled back.

The route takes the document as JSON. `blizzard runner settings apply FILE` reads a TOML file and decodes it through the
host loader's decode step (`runner/host_config.py`), the one runner module `bzh:config-key-declared`'s ast-grep rule
lets read TOML, and sends the JSON; `settings export` asks for the JSON and emits TOML through `foundation/toml_emit.py`
([hub.md](./hub.md) §Apply). No new decoding seam is opened, and the rule is unchanged.

The answer lists one outcome row per entry and record, as the hub's apply does: `created`, `edited` with the keys it
moved, `enabled`, `unchanged`, or `refused` with its errors. `GET /api/host/settings/export` writes the store as an
apply document, so applying an export back writes nothing, and an operator who keeps their host's settings in version
control exports, edits, and applies.

## CLI

Verbs under `blizzard runner`, each reaching the host through `RunnerDaemon.reach` (`runner/cli/daemon.py:54`): the unix
socket, or `--runner-url`. Each is declared in `contracts/cli/runner.json`.

| Verb                                                    | Route                                                                                 |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `settings show [KEY] [--runner NAME] [--host]`          | the view; one key's row with `KEY`                                                    |
| `settings set KEY VALUE [--runner NAME] [--host]`       | a write; `VALUE` parsed as a TOML value, so a table or list is written as TOML inline |
| `settings unset KEY [--runner NAME] [--host]`           | a write                                                                               |
| `settings changes [--runner NAME] [--host]`             | the change log                                                                        |
| `settings apply FILE [--dry-run]`                       | an apply                                                                              |
| `settings export`                                       | the export, as TOML on stdout                                                         |
| `workspace create\|list\|show\|edit\|retire\|enable`    | the workspace routes                                                                  |
| `subscription create\|list\|show\|edit\|retire\|enable` | the subscription routes                                                               |

`--runner` takes a runner's name or id and defaults to the host's only runner, refusing to guess between several, as the
existing operator verbs do. A `set`, `unset`, or `edit` reads the current revision and sends it as `If-Match`, so two
operators editing at once are refused rather than overwriting each other. `runner add` and `runner retire` keep their
names and learn the running host ([host-records.md](./host-records.md) §Runners).

## The local page

The runner's Angular app (`web/projects/runner`) gains a **Settings** tab beside Board, Events, and Host:

- **Host settings** and, per hosted runner, **Runner settings**: every key of the view, grouped by its table, each
  showing its value, its source, and its lifetime as a badge an operator reads as "when will this bite". The `in_force`
  line sits at the top: "running revision 6 since 22:14 — revision 7 written, takes hold at the next tick".
- An edit dialog per key for the superuser, sending a one-key write; a refused write shows each error beside its key. A
  `boot` key is not shown.
- **Workspaces** and **Subscriptions**: lists of the records with their state, a detail with the six verbs, and the
  record's change history.
- **History**: the change log, each row showing its door, actor, revision, and the keys it moved, old and new.

A session without the superuser role sees every page and no edit control.
