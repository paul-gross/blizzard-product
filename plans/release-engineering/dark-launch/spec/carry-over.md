# Carry-over

A fleet that predates this epic has no flags, so nothing it runs changes when the epic lands: the registry arrives
empty, every process resolves an empty `Flags`, and every answer this epic adds reads as nothing on.

## The order of landings

1. **The registry and its reading.** `foundation/flags/`, the empty `registry.toml`, the `[flags]` table in both
   configuration files, `GET /api/flags` on the hub and on the host, `overridden` on `GET /api/health`, and the registry
   lint ([registry.md](./registry.md), [reading.md](./reading.md)). Nothing reads a flag yet, so it lands alone.
2. **The fleet scope.** `fleet_flags` on `RunnerIdentityView`, the runner applying it at the head of its tick, and
   `mock_hub`'s lever for it, in `blizzard` and `blizzard-mock` together. A runner from before this step meets a hub
   that answers the field and ignores it; a runner after it meets a hub from before and reads every fleet flag off,
   which is the behavior that hub carries.
3. **Surface behind a flag.** `FlagOff` in `wire/`, the CLI's refusal, the configuration warning for a key whose flag is
   off, the board's flag store, guard, and marker, and the rule `bzh:flag-hides-behavior` in `blizzard-context`
   ([contract.md](./contract.md)).
4. **The lifecycle check.** `blizzard-flag-lifecycle` in the merge gate ([lifecycle.md](./lifecycle.md)). It reads
   release tags, so its release rules have nothing to hold until `epic:release-engineering/first-release` has cut one,
   and its supported-release rules read `epic:release-engineering/compat-floor`'s baseline once that lands; until then
   it checks structure and creation alone. The rule tying a flip to its change note waits for the note format of
   `epic:release-engineering/notes`. Its run at the cut joins the release procedure of
   `epic:release-engineering/release-lines`, in whichever of the two epics lands second.
5. **The procedure.** The workflow in `blizzard-context`, the check's row in `verification/blizzard.md`, and
   `docs/flags.md`, written against what the four steps above built.

The first flag declared after step 3 is the proof: the next piece of whichever epic is landing dark at the time, carried
through the procedure by an agent told only its name.

## What stays as it is

The modes `blizzard-hub.toml` already holds — `route_token_mode` and `produces_mode`, each `warn` or `enforce`
(`hub/config.py`) — are not flags and do not move. Each is an operator's standing decision about how strictly the hub
holds a rule, with no release that retires it; a flag is the release's own decision, gone once a stable release after
the one that made it has shipped. The same holds for every configured record of `epic:live-config`: what an operator
chooses for their fleet stays theirs to choose, and a flag never becomes one.

Work already on `master` and unfinished when this epic lands is not retrofitted behind a flag. It finishes as it would
have, inside the version rules `epic:release-engineering/compat-floor` holds it to; only work started after step 3 lands
dark.

## The dogfood instance

The hosted hub's `blizzard-hub.toml` lives in `blizzard-infra`, and the runner host's `blizzard-host.toml` in the
workspace's runner directory. Neither carries a `[flags]` table unless its owner decides to run a piece of work early;
when one does, the table changes by a reviewed change to the repository that holds the file, like any other change to
the instance, and both the board and the runner's panel show it is running an experiment.
