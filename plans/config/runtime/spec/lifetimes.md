# Lifetimes in the registry

How a setting declares its lifetime, how an environment variable is tied to the one kind of setting it may supply, and
the one rule every pass follows when it evaluates its settings. The lifetimes themselves and every key's lifetime are
[settings.md](./settings.md); this file is the machinery that carries them.

## The key declares it

`ConfigKey` (`foundation/config_keys.py:49`) gains two fields:

| Field      | Type               | Rule                                                                                                                                                                                                                                                |
| ---------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lifetime` | `Lifetime \| None` | One of `PASS`, `SESSION`, `BOOT`, `IMMUTABLE` — a `StrEnum` in `foundation/config_keys.py` whose values are the lowercase names. Required on every key that holds a value, and absent on every key that only groups others (§Which keys carry one). |
| `env`      | `str \| None`      | The environment variable that supplies the key's value, where one does (§Environment variables). Legal only on a `BOOT` key.                                                                                                                        |

### Which keys carry one

A key that holds a value carries a lifetime: every type but `TABLE` and `ARRAY_OF_TABLES`, an `OPEN_TABLE` such as
`hub_tokens` or `models.aliases` included, since its whole value is one setting. A `TABLE` or `ARRAY_OF_TABLES` key —
`tracing`, `auth`, `runner`, `workspace`, `workspace[].repo` — only groups its children and carries none; its children
carry theirs. `ConfigRegistry` fails its construction on a value key with no lifetime, a grouping key with one, or a
non-`BOOT` key with `env`, so a key cannot be added without its lifetime being decided.

A `deprecated` key keeps a lifetime, since it is still read — to be honored, refused, or graduated — and it is the
lifetime of the key that replaces it. A key read at boot only to be refused, such as the hub's `work_source[]` — read
now only so the hub can refuse to start and name the import that moves it into records — is declared `deprecated`, with
`replaced_by` naming the work-source records, and its lifetime is `boot`, since its replacement is a record kind rather
than a key and boot is when it is read.

### Environment variables

`env` names a variable that supplies the key's value and overrides the file, as each of these does today:

| Key          | Daemon | `env`           |
| ------------ | ------ | --------------- |
| `host_token` | runner | `BZ_HOST_TOKEN` |
| `host`       | hub    | `BZ_HUB_HOST`   |
| `port`       | hub    | `BZ_HUB_PORT`   |
| `db_url`     | hub    | `BZ_HUB_DB_URL` |

Nothing else is declared through `env`:

- **A key whose value names a variable** — `session_secret_env`, `egress.path_key_env`, an OAuth provider's
  `client_secret_env` — is an ordinary `boot` key. The variable it names is read at boot, but it supplies a secret, not
  the key.
- **A token file's variables** keep their `EnvFile` declaration (`runner/config_keys.py:286`), which describes the
  family `BZ_HUB_TOKEN_<NAME>` that supplies `hub_tokens`; both are deprecated, read only by the seed that imports each
  runner's token into the store ([runner-store.md](./runner-store.md) §Sealed tokens).
- **A variable that supplies no key** — `BZ_RUNNER_DIR`, the CLI's runtime root (`runner/cli/env.py:4`), and
  `BZ_HUB_SECRET_KEY`, the hub's sealing key (`hub/secrets/internal/env_keys.py:13`), and `BZ_HOST_SECRET_KEY`, the
  host's ([runner-store.md](./runner-store.md) §Sealed tokens) — and OpenTelemetry's own variables stay outside the
  registry: they are not settings.

This table is the one list of variables that supply a setting; [settings.md](./settings.md) §Environment variables
points here.

## The contract carries it

`tools/config_contract.py` writes `lifetime` and `env` on every entry of `keys[]` in `contracts/config/hub.json` and
`contracts/config/host.json`, beside `type` and `default`; `env` is written as `null` where a key has none. A lifetime
changed without a regenerated document turns the `contract drift` step red, and `blizzard:config-contract` holds the
registries to the documents. `contracts/config/README.md` gains the two fields and the four values.

Adding a field to every key is additive to the document (`bzh:released-surface-additive`). Moving a key's lifetime later
in the order `pass`, `session`, `boot`, `immutable` narrows what an operator can change without a restart, and is a
break; moving it earlier is additive. `docs/versioning.md` §What counts as breaking gains the row, and `blizzard:compat`
gains the check: a key whose lifetime in the candidate's document comes later than in a supported release's is reported
as a break, as a removed key is.

## Entries

A key whose lifetime is `pass` or `session` is an **entry** of its daemon's store, and an `immutable` key is an entry
written once at creation. A `boot` key is never an entry. The strict accessor (`ConfigView`) keeps serving `boot` keys
from the file; an entry is read only through its daemon's settings reader — [runner-store.md](./runner-store.md) §The
reader for the runner, [hub.md](./hub.md) §Reading per pass for the hub — and `bzh:config-key-declared` gains the
sentence that says so: a `pass` or `session` key read through `ConfigView` outside the seed is a finding. The sentence
lands for each daemon in the piece that makes the last of its keys an entry ([carry-over.md](./carry-over.md)): the
runner's with piece 4, the hub's with piece 9.

## Evaluated at the start of a pass

Every pass of either daemon evaluates its settings the same way:

1. At its start, before any other work, the pass reads one **snapshot**: the revisions of every settings record it
   reads, and the entries themselves when a revision has moved since the snapshot it last built from. One indexed read
   of the revisions per pass is the steady state (`bzh:live-set-read`).
2. Anything built from an entry — a hub client's timeout, a harness adapter's alias table, a sampler — is built from the
   snapshot and kept under the revisions it was built from, reused while they hold, rebuilt when they move
   (`bzh:config-read-on-use`).
3. The pass works from that snapshot to its end. Nothing in it reads an entry again, so a write that commits while the
   pass runs reaches the next one.
4. A worker launched by the pass is handed the pass's snapshot of every `session` key, and keeps it for its session
   ([passes.md](./passes.md) §Sessions).

A hub request is a pass of its own: it takes its snapshot when it arrives. Neither daemon watches its store for changes
and pushes them into running objects; the next pass finding a moved revision is the only way a change arrives. That is
what keeps a cold start and a long-lived process equally correct.

`bzh:config-read-on-use` is amended to bind entries as it binds configured records — the runner's in piece 3, the hub's
in piece 9, which also narrows its scope sentence on the hub's process settings to the `boot` keys. A new rule beside
it, `bzh:settings-snapshot-per-pass`, lands with piece 3 binding the runner, and piece 9 extends its scope to the hub.
It states the four steps above, its detect list naming an entry read after a pass has begun, an entry captured into an
object at composition, and a `pass` or `session` key read from the environment.
