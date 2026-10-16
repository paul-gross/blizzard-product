# A flag's life

## Read from the releases

A flag's age is never written down. `blizzard-flag-lifecycle` (`tools/flag_lifecycle.py`, run by
`mise run flag-lifecycle` and declared as `blizzard:flag-lifecycle` in `blizzard-context`'s `verification/blizzard.md`)
reads `registry.toml` at `HEAD` and at every release tag, newest first, and derives for each name:

| Derived    | Read as                                                                                 |
| ---------- | --------------------------------------------------------------------------------------- |
| Born in    | The oldest release, candidate or stable, whose registry holds the flag live             |
| On in      | The oldest stable release whose registry holds it live with `default = "on"`            |
| Due        | Once a stable release later than *on in* has shipped with the flag still live           |
| Retired in | The oldest release whose registry holds it in `[retired]`                               |
| Off for    | The number of releases, candidates and stable alike, since *born in* that held it `off` |

Release tags are the ones `epic:release-engineering/first-release` cuts and `epic:release-engineering/release-lines`
names. Only a stable release settles when a flag turned on: a stable release is tagged at the same commit as the
candidate it promotes, so a flag the candidate carried on is on in that stable release, and counting the candidate would
leave the flag due before its own stable release shipped. Candidates still count toward a flag's age, since the fleet
runs them. Before the first release there are no tags and the check passes on structure alone.

## What it holds

| The check finds                                                           | On a pull request | At the cut of a candidate, and on a release tag |
| ------------------------------------------------------------------------- | ----------------- | ----------------------------------------------- |
| A flag created with `default = "on"`                                      | fails             | fails                                           |
| A flag set back from `on` to `off` after a stable release carried it on   | fails             | fails                                           |
| A flag whose *on in* is a stable release, still live                      | warns             | —                                               |
| A flag that is due, still live                                            | warns             | fails                                           |
| A flag off for twelve releases — the work has stalled                     | warns             | warns                                           |
| A tombstone no supported release carries live — it may go                 | notes             | notes                                           |
| A tombstone removed while a supported release still carries the name live | fails             | fails                                           |
| A name reused after it was retired, at any tag                            | fails             | fails                                           |
| A change setting a flag `on` with no change note naming it                | fails             | —                                               |

The due rule is what makes a flag last one stable release on, with one more for its retirement to land: the stable
release that turns it on carries it, the next stable release may carry it still while its retirement is written, and
every cut after that refuses it. A candidate that turned a flag on may set it back off before its stable release ships,
since no stable release has promised the capability yet. The supported releases are
`epic:release-engineering/compat-floor`'s, read through its baseline rather than recomputed here. The check runs in the
merge gate on every pull request, and the release procedure of `epic:release-engineering/release-lines` runs it again at
the cut, before anything is tagged.

## Turning a flag on

Turning a capability on is one change: the registry's `default` set to `on`, and the change note that describes the
capability to an operator, which names the flag. The note's format, where it lives, and how the release's notes are
assembled from it are owned by `epic:release-engineering/notes`; this epic asks of it only that a note can name the
flags it turns on, and that the assembled notes list every flag whose *on in* is the release being cut, under the
summary the registry gives it. The two are one change because the release that turns a flag on is the release whose
notes describe it, and a reviewer reads them side by side.

## Retiring a flag

Retiring deletes the path that was off: every `flags.on(name)` branch collapses to its on side, every `FlagOff` answer
for the flag goes with it, the board's guard and markers go, and the name moves to `[retired]`. What the flag guarded
stays exactly as it was released ([contract.md](./contract.md)). A retirement carries no change note of its own, since
nothing an operator sees changes; an override of a retired flag now only warns ([reading.md](./reading.md)).

## Testing both sides

The merge gate runs the suite with every flag at its registry default, so the default is always the proven fleet. A
flag's on path is proven by tests that build their `Flags` with it on — unit and service tiers through composition, the
upper tiers through the `[flags]` table of the configuration they already write — and those tests live with the work
from the change that adds the flag. Turning a flag on moves its on-path tests onto the default path without change, and
the change that turns it on keeps every off-path test until the flag retires, so both sides stay proven for the release
that carries both. A fleet flag's tests prove the hub and a runner agreeing through the identity answer, against
`blizzard-mock`'s lever for the runner and its driver for the hub.

## The procedure

`blizzard-context` gains one procedure under `workflows/`, beside `workflows/release.md`, in three parts an agent can be
told to run by name: landing a piece of work behind a new or existing flag — choosing its scope, declaring it, guarding
the behavior and only the behavior, proving the on path; turning a flag on — the one change, its note, the proof that
the on path is now the gate's; and retiring a flag once it is due. The rule it leans on, `bzh:flag-hides-behavior`,
lives under `standards/` with the table of what each surface does while off ([contract.md](./contract.md)).
