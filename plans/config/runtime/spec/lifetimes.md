# Lifetimes in the registry

How a setting declares its lifetime, how an environment variable is tied to the one kind of setting it may supply, and
the one rule every pass follows when it evaluates its settings. The lifetimes themselves and every key's lifetime are
[settings.md](./settings.md); this file is the machinery that carries them.

## The key declares it

`ConfigKey` (`foundation/config_keys.py:49`) gains two fields:

| Field      | Type                 | Rule                                                                                                                                                                                                    |
| ---------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lifetime` | `Lifetime`, required | One of `PASS`, `SESSION`, `BOOT`, `IMMUTABLE` — a `StrEnum` in `foundation/config_keys.py` whose values are the lowercase names. No default: a key added without one fails the registry's construction. |
| `env`      | `str \| None`        | The environment variable that supplies the key, where one does. Legal only on a `BOOT` key: a `ConfigRegistry` holding a non-`BOOT` key with `env` set fails its construction.                          |

A `deprecated` key keeps a lifetime, since it is still read — to be honored, refused, or graduated — and its lifetime is
the one its replacement has. A table key whose children differ declares each child; the registry refuses a table key
carrying a lifetime when any child declares its own.

Both registries (`blizzard.hub.config_keys`, `blizzard.runner.config_keys`) give every key its lifetime from the tables
in [settings.md](./settings.md) in one change. The variables that already supply a `boot` value outside the registry —
`BZ_RUNNER_DIR`, `BZ_HOST_TOKEN`, `BZ_HUB_SECRET_KEY`, the `egress.path_key_env` variable — are declared through `env`
on the key they supply, so the contract names them. A token file's variables keep their `EnvFile` declaration
(`runner/config_keys.py:286`), which already describes a family of names rather than one. OpenTelemetry's own variables
stay undeclared: they configure a library, not a blizzard key.

## The contract carries it

`tools/config_contract.py` writes `lifetime` and `env` on every entry of `keys[]` in `contracts/config/hub.json` and
`contracts/config/host.json`, beside `type` and `default`; `env` is written as `null` where a key has none. A lifetime
changed without a regenerated document turns the `contract drift` step red, and `blizzard:config-contract` holds the
registries to the documents. `contracts/config/README.md` gains the two fields and the four values.

Adding a field to every key is additive to the document (`bzh:released-surface-additive`). A key's lifetime moving from
`pass` or `session` to `boot` is a narrowing an operator relies on and is judged as a break (`bzh:judging-a-break`); a
move the other way, from `boot` to `pass`, is additive.

## Entries

A key whose lifetime is `pass` or `session` is an **entry** of its daemon's store, and an `immutable` key is an entry
written once at creation. A `boot` key is never an entry. The strict accessor (`ConfigView`) keeps serving `boot` keys
from the file; an entry is read only through its daemon's settings reader — [runner-store.md](./runner-store.md) §The
reader for the runner, [hub.md](./hub.md) §Reading per pass for the hub — and `bzh:config-key-declared` gains the
sentence that says so: a `pass` or `session` key read through `ConfigView` outside the seed is a finding.

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
what keeps a cold start, a second hub process, and a long-lived one equally correct.

`bzh:config-read-on-use` is amended to bind entries as it binds configured records, and its scope sentence on process
settings narrows to the `boot` keys. A new rule beside it, `bzh:settings-snapshot-per-pass`, states the four steps
above, its detect list naming an entry read after a pass has begun, an entry captured into an object at composition, and
a `pass` or `session` key read from the environment.
