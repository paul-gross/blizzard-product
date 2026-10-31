# Resolving a flag

## Overrides

An override is one table, `[flags]`, in the configuration file a process already reads: `blizzard-hub.toml` for the hub
(`hub/config.py`), `blizzard-host.toml` for a host and its runners (`runner/host_config.py`). Its keys are flag names
and its values `true` or `false`:

```toml
[flags]
toolkit-dialogs = true
```

| The key names                | The process                                                                                           |
| ---------------------------- | ----------------------------------------------------------------------------------------------------- |
| A live flag of its own scope | Applies it                                                                                            |
| A `fleet` flag, on a host    | Ignores it and logs a warning naming the hub as the flag's owner (§The fleet scope across skew)       |
| A `hub` flag, on a host      | Ignores it and logs a warning; a `runner` flag on the hub, likewise                                   |
| A tombstone                  | Ignores it — the behavior is always on — and logs a warning that the key can be removed               |
| No flag at all               | Ignores it and logs a warning naming the key, so a typo is seen without failing a deployment over one |

A value that is not a boolean fails the load, as any malformed key does. `[flags]` is an open table: its keys are not
configuration keys under `docs/versioning.md`, adding or removing a flag never changes the configuration surface, and
the configuration diff of `epic:release-engineering/compat-floor` reads the table's presence, never its keys.

Overrides are read once, at start, with the rest of the file; changing one is a restart. There is no environment
variable, no hub record, and no admin door for a flag, and none is per tenant.

## The hub

The hub resolves its `Flags` at composition: every `hub` and `fleet` flag at its registry default, then its `[flags]`
table. The answer holds for the process's life and is the same for every tenant it serves.

The hub publishes it on a public-plane route, `GET /api/flags`, beside `GET /api/health` (`hub/api/health.py`): an open
map of every live `hub` and `fleet` flag to whether it is on, and the list of those its `[flags]` table changed.

```json
{ "flags": { "toolkit-dialogs": true }, "overridden": ["toolkit-dialogs"] }
```

The response model is `dict[str, bool]`, never an enumeration of names, so adding or retiring a flag never changes the
hub's OpenAPI document (`openapi/hub.openapi.json`). `GET /api/health` gains `overridden: list[str]`, so anything that
already reads the hub's version also sees that it is running an experiment.

## The board

The board ships with the hub that serves it (`static/hub/`), so it never meets a flag its hub does not know. The shell
reads `GET /api/flags` once at start through the generated client and holds the answer in one injectable store
(`web/projects/hub/src/app/shell/`), which features read as a signal — `flags.on('toolkit-dialogs')` — and which a route
guard reads to keep an unfinished screen out of the navigation. A flag name the answer does not hold reads as off, since
the board cannot fail a page on a name; the registry lint covers the board's names as it covers Python's. The board
renders a quiet marker beside the hub's version whenever `overridden` is not empty.

## A runner

A host resolves one `Flags` for each of its runners: every `runner` flag at its registry default, then the host's
`[flags]` table, then every `fleet` flag as its hub last answered it (below). The runner's panel reads its host's answer
from the host's own `GET /api/flags`, shaped as the hub's over every live `runner` flag, and shows the same marker when
its host overrides anything. The host carries `overridden` on that answer alone; its `GET /api/health` is unchanged.

## The fleet scope across skew

A `fleet` flag is the hub's to answer, because only the hub can make every runner agree. `RunnerIdentityView`
(`wire/runner.py`), which every runner reads at the head of every tick (`runner/lifecycle/identity_check.py`), gains an
open map:

```python
#: Every live ``fleet`` flag the hub resolves, and every ``fleet`` tombstone, as on. Empty from a hub that predates flags.
fleet_flags: dict[str, bool] = {}
```

A runner applies the latest answer to its `fleet` flags before the rest of the tick runs, so a hub restarted with a flag
turned on reaches every runner within one tick, and none of them ever acts on a fleet flag its hub has off. A runner
reads a `fleet` flag the answer does not name as off: either the hub predates flags, or predates the flag itself, and in
both cases the hub does not carry the behavior.

The skew window keeps the rest sound. A runner never runs newer than its hub, so the hub always knows every fleet flag a
runner does; a hub newer than its runner may have retired one the runner still holds live, and its tombstone answers it
on, which is the behavior the hub now always has. A hub newer than its runner may also hold fleet flags the runner has
never heard of; the runner ignores them, and the hub keeps serving the runner's released behavior, as the window already
requires of it. The field is additive to the wire under `bzh:fleet-wire-additive`, and `blizzard-mock` answers it empty.
