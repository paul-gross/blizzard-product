# Carry-over

A fleet that predates this epic has no flags, so nothing it runs changes when the epic lands: the registry arrives
empty, every process resolves an empty `Flags`, and every answer this epic adds reads as nothing on.

## The order of landings

1. **The registry and its reading.** `foundation/flags/`, the empty `registry.toml`, the `[flags]` table in both
   configuration files, `GET /api/flags` on the hub and on the host, `overridden` on the hub's `GET /api/health`, and
   the registry lint over Python ([registry.md](./registry.md), [reading.md](./reading.md)). Nothing reads a flag yet,
   so it lands alone. `[flags]` is one entry of type `open table` in `epic:release-engineering/compat-floor`'s
   configuration key registries, whose strict mode would otherwise refuse it; whichever of the two lands second declares
   the entry.
2. **The fleet scope.** `fleet_flags` on `RunnerIdentityView`, the runner applying it at the head of its tick, and
   `mock_hub`'s lever for it, in `blizzard` and `blizzard-mock` together. A runner from before this step meets a hub
   that answers the field and ignores it; a runner after it meets a hub from before and reads every fleet flag off,
   which is the behavior that hub carries.
3. **Surface behind a flag.** `FlagOff` in `wire/`, the route guard, the CLI's refusal, the configuration warning for a
   key whose flag is off, and the rule `bzh:flag-hides-behavior` in `blizzard-context` ([contract.md](./contract.md)),
   in `blizzard` and `blizzard-context` together.
4. **The board and the host panel.** Each app's flag store, route guard, and override marker, and the registry lint's
   scan of `web/projects/` ([reading.md](./reading.md) §The board). It needs only step 1, so it lands beside step 3 in
   either order.
5. **The lifecycle check, on its own rules.** `blizzard-flag-lifecycle` with its three modes, the derivation from
   release tags, every rule that reads only the registry and the tags, and `gate / flag lifecycle` on pull requests
   ([lifecycle.md](./lifecycle.md)). Its tag rules are proven against a synthetic history of tags in the tool's tests,
   so it needs no release to land, and with no tag holding a registry it checks structure alone.
6. **The lifecycle check, against its siblings.** The tombstone rules, read through `blizzard-compat releases` of
   `epic:release-engineering/compat-floor`; the rule tying a flip to a note whose `flag` field names it, in the format
   of `epic:release-engineering/notes`; and the calls into the release: `cut` mode in the gate on a pull request whose
   head carries the `Release-Cut` trailer, and `tag` mode in `release.yml` on every `v*` tag before anything is
   published, the stable tag's borrowed-proof run included. `epic:release-engineering/release-lines` defines the cut
   commit and the borrowed proof; this epic adds the check's calls to both. The step waits for all three siblings, and
   so behind `epic:release-engineering/first-release`'s cut of the first candidate, since compat-floor's check lands
   only once a release at or above its floor exists.
7. **The procedure.** The workflow in `blizzard-context`, the check's row in `verification/blizzard.md`, and
   `docs/flags.md`, written against what the steps above built.

The first flag declared after steps 3 and 4 is the proof: the next piece of whichever epic is landing dark at the time,
carried through the procedure by an agent told only its name.

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
