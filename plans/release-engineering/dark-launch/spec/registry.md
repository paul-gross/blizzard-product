# The flag registry

## The file

Every flag is declared in one data file, `foundation/flags/registry.toml`, packaged into the wheel and the image with
the rest of `blizzard` and read by the hub, every runner, and the lifecycle check alike. It is data rather than Python
so that the check can read it at any release tag with `git show <tag>:src/blizzard/foundation/flags/registry.toml`,
never importing an old release's code ([lifecycle.md](./lifecycle.md)).

```toml
[flag.toolkit-dialogs]
owner = "epic:ui-toolkit"
summary = "Every board dialog assembled from the component toolkit"
scope = "hub"
default = "off"

[retired]
subscription-brake-fanout = "fleet"
```

| Field     | Rule                                                                                                                                                                              |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name      | The table key: lowercase kebab-case, at most 48 characters, unique across `[flag.*]` and `[retired]` for the life of the file — a name is never reused once retired               |
| `owner`   | The `epic:` or `initiative:` id the work belongs to, resolvable in `blizzard-product`. A flag with no epic behind it is a bug fix that should not be dark                         |
| `summary` | One line in the operator's words, read on the panel and in the notes when the flag turns on. It describes the capability, never the implementation                                |
| `scope`   | `hub`, `runner`, or `fleet` (below)                                                                                                                                               |
| `default` | `off` or `on`. Created `off`; set `on` only by the change that turns the capability on in a release, which the lifecycle check holds to its note ([lifecycle.md](./lifecycle.md)) |

Nothing in the file is a version or a date. When a flag was added, when it turned on, and when it is due to retire are
read from the release tags that carried it, so no author ever guesses which release their change will ride.

## Scopes

| Scope    | What it changes                                                                    | Whose answer counts                                                                                                 |
| -------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `hub`    | Behavior the hub alone carries out — its routes, its actions, the board it serves  | The hub's ([reading.md](./reading.md) §The hub)                                                                     |
| `runner` | Behavior a runner carries out alone, with no new expectation of the hub            | The runner's, resolved by its host ([reading.md](./reading.md) §A runner)                                           |
| `fleet`  | Behavior the hub and its runners must agree on — a new fact kind, a new claim step | The hub's, carried to every runner on its identity answer ([reading.md](./reading.md) §The fleet scope across skew) |

A flag whose behavior is split across the hub and a runner is `fleet`, never a `hub` flag and a `runner` flag that have
to be turned on together. Choosing the scope is part of declaring the flag, and the review of the change that adds it
judges it.

## Tombstones

Retiring a flag moves its name from `[flag.*]` to `[retired]`, keeping its scope. A tombstone answers `on` wherever it
is asked, so a runner older than its hub that still knows the flag as live hears the behavior its hub now always has,
and a configuration file that still overrides it meets a warning rather than an unknown key. A tombstone stays until no
supported release carries the name as a live flag, and the lifecycle check says when it may go.

## Reading a flag

Code reads a flag through one call and never parses the file or the configuration itself:

```python
from blizzard.foundation.flags import Flags

if flags.on("toolkit-dialogs"):
    ...
```

`foundation/flags/` holds the registry's loader, the frozen `FlagRegistry` it yields, and `Flags`, the resolved answer
for one process: the registry's defaults with that process's overrides and, for a runner, its hub's fleet answers
applied ([reading.md](./reading.md)). `Flags` is built at composition and handed to whatever needs it through the same
holders configuration already travels by (`hub/composition.py`, `runner/composition.py`), never read from a module-level
global, so a test builds the `Flags` it wants. `flags.on(name)` raises on a name the registry does not hold, live or
retired, so a mistyped name fails its first test rather than reading as off. The registry file is the single home of
every name; a lint (`tests/test_flag_registry.py`) fails when code names a flag the file does not hold, or the file
holds a live flag no code reads.
