# Carry-over

The epic lands on a runner host that reads its file once and freezes it, whose loop ticks on an environment variable,
and whose runners, workspaces, and subscriptions are fixed at its start; and on a hub whose operational settings are
read once from its file. It leaves a host whose every runner setting is changed through its API and takes hold at the
next pass or the next session, whose runners, workspaces, and subscriptions come and go while it runs, and whose file
seeds what its store lacks; and a hub whose operational settings are one hub-wide record changed the same way. No
setting's name changes, no running session is ever disturbed by a change, and a host or hub that is never edited through
its API behaves as it did, once its file has seeded it.

## The order of landings

Each piece lands on its own, leaves `master` green, and is proven by the methods it names. A piece that changes
`blizzard-context` lands that half there, proven by `blizzard-context:reference-lint`,
`blizzard-context:markdown-format`, `blizzard-context:markdown-lint`, and `blizzard-context:registry-drift`, and no half
cites a rule that has not landed. Every piece that changes an operator surface regenerates its contract documents,
carries its change note (`blizzard:change-notes`), and updates the operator pages it touches. No piece changes the fleet
wire, so none extends `blizzard-mock`. Piece 1 is pre-work. The runner half is pieces 2 to 8; the hub half, 9 and 10,
waits on piece 3, whose rule it extends to the hub. The epic needs both.

1. **The sealing, shared.** Waits for nothing. Pre-work: the hub's AES-256-GCM sealing, key material, key providers, and
   rotation move from `hub/secrets/` to `foundation/secrets/`, the hub's keys, ciphertexts, and behavior unchanged
   ([runner-store.md](./runner-store.md) §Sealed tokens). No operator surface changes. Proven by `blizzard:unit-test`
   and `blizzard:component-test` over the hub's secrets as they stand, unchanged, and `blizzard:structural-gate` for the
   new package's layering.
2. **Lifetimes in the registry.** Waits for nothing. `Lifetime` and `ConfigKey.lifetime` and `env`, every key of both
   registries given its lifetime from [settings.md](./settings.md), the variables that supply a key declared, both
   contract documents gaining the two fields, the hub's `work_source[]` declared `deprecated`
   ([lifetimes.md](./lifetimes.md)). The host file's `tick_seconds` key, `pass`, read at boot for now;
   `BZ_RUNNER_TICK_SECONDS` deprecated, honored only when the file states none; both drivers taking an interval
   callable. Every test and script that sets the variable moves to the file key in the same change: the crash suite's
   support (`tests/crash/support.py:59`), the runner and journey service specs, the two store-migration service tests,
   the rehearsal installer, the runner-host harness check, the step-pass component test, and the OpenCode bundle probe.
   `docs/versioning.md` §What counts as breaking gains the lifetime row, and `blizzard:compat` reports a key whose
   lifetime comes later than in a supported release's document ([lifetimes.md](./lifetimes.md) §The contract carries
   it). Its context half: `bzh:config-key-declared` names the lifetime and the `env` rule. Proven by
   `blizzard:unit-test` over a registry refusing a value key with no lifetime, a grouping key with one, and a non-`boot`
   key with `env`; `blizzard:compat` against a fixture document moving a key from `pass` to `boot`;
   `blizzard:config-contract`; `blizzard:harness-check`; `blizzard:crash-sweep` on the file key.
3. **A runner's settings, live.** Waits for piece 2. The runner-scoped settings tables and their revision, each read
   through the runner's bound connection; the seed of each runner's keys, with its identity row; the reader; the tick's
   evaluation and the context's rebuild; the harness adapters, telemetry plan, and health probes taking their settings
   at the call, the version cache keyed by binary; `SessionSettings` handed to each launch; the spawn's revisions and
   `transcripts_root`; the superuser check; the runner's settings routes and their `settings` CLI verbs without
   `--host`, `apply`, or `export` ([runner-store.md](./runner-store.md), [runner-api.md](./runner-api.md),
   [passes.md](./passes.md) §A runner's tick, §The objects settings reach, §Sessions). Each per-runner key becomes an
   entry in this piece, so no key is ever ignored in the file without a way to change it. The test harness's
   `reconfigure()` (`tests/harness/runner-host.ts:168`) writes through the settings API on the running host instead of
   editing a stopped host's file, and every spec that reconfigured a host — among them
   `tests/harness/checks/runner/host-capabilities.spec.ts` and `tests/service/runner/runner.spec.ts` — is updated to it,
   any that only proved the file was re-read at boot deleted, and new specs added for what the API now promises. The
   operator page on runner configuration gains entries, lifetimes, and the seed; its upgrade note says the file now
   seeds a runner's keys and is not read for them again. Its context half: `bzh:settings-snapshot-per-pass`, binding the
   runner; `bzh:config-read-on-use` binding the runner's entries. Proven by `blizzard:unit-test` over every refusal of a
   write and over the seed's three cases, an unset entry surviving a restart against a file stating the key;
   `blizzard:component-test` over a write landing between two ticks and taking hold at the second, a write during a tick
   not reaching it, a refused write leaving the runner as it was, two writes racing one revision, lowering `agents`
   below the running sessions stopping claims and no session, a model alias, autonomy, and worker-environment edit each
   reaching the next spawn and not a running one, a moved `claude_code.binary` probed at the next refresh, a session's
   transcript read from the root it launched with after `transcripts_root` moves, and a rebuild that does not re-run the
   scratch sweep; `blizzard:service-test` on a stepped host writing `throttle.budget.usd` and stepping the tick;
   `blizzard:harness-check` over the new `reconfigure()`; `blizzard:store-contract`; `blizzard:cli-contract`;
   `web:client-drift`; `blizzard:compat`; `blizzard:rehearsal`, an upgraded host seeded from its file running exactly as
   before; and `blizzard:manual-runner`.
4. **The host's settings, live.** Waits for piece 3. The host-level tables, their seed, the host tick's and trace
   sweep's evaluation, `tick_seconds` and `tracing.sweep_seconds` read live with the wake on write, `--host` on the CLI
   ([passes.md](./passes.md) §The trace sweep, §The loop's interval). Its context half: `bzh:config-key-declared` gains
   the runner's half of the sentence refusing a `pass` or `session` key read through `ConfigView`, now that every runner
   key of those lifetimes is an entry. Proven by `blizzard:unit-test` over the driver's wake running at once or waiting
   out the rest; `blizzard:component-test` over a `tick_seconds` write from an hour to a second taking hold without
   waiting the hour, and a tracing write reaching the next sweep; `blizzard:harness-check`; `blizzard:store-contract`;
   `blizzard:cli-contract`; `web:client-drift`; `blizzard:compat`.
5. **Runners on a running host.** Waits for pieces 1 and 4. The host-scoped hosted-runner tables and their seed; the
   sealed token table, the host's key and its key directory, and the seed's import of every runner's token; the add and
   retire routes; the host tick's reconcile; a runner's drain alone; `RetiredRunnerWatch` writing its fact; the app's
   settings read from the host rather than its first runner; `runner add` and `retire` reaching a running host and
   writing the store on a stopped one; `.env`'s runner tokens, `[hub_tokens]`, and `runners/<name>/` deprecated; ingest
   writing the destination's hosted row and re-sealing the moving runner's token ([host-records.md](./host-records.md)
   §Runners, [runner-store.md](./runner-store.md) §Sealed tokens). Its context half: `bzh:crash-exemptions-runner`
   records the add's window; `runner-scoping.md` and `crash-correctness/runner.md` say a runner's binding is made from
   `hosted_runners`. The operator pages on adding runners, on the host's runtime directory, and on its credentials
   change to match. Proven by `blizzard:unit-test` over a token sealed under one runner's id refusing to open under
   another's; `blizzard:component-test` over an add starting the runner at the next tick and leaving its neighbours'
   ticks unbroken, a forced retire draining one runner's sessions and no other's, the host's first runner retired with
   the page still served, a hub-side retirement stopping the runner's threads, a rename leaving the runner's token
   found, and an upgraded host whose tokens were in `.env` and `[hub_tokens]` finding every one in the store;
   `blizzard:service-test` adding a runner to a stepped host and claiming with it; `blizzard:crash-sweep` with crash
   points inside the add and the token import, the invariant checker passing at each; `blizzard:component-test` over
   ingest moving a runner whose token re-seals; `blizzard:store-contract`; `blizzard:config-contract`;
   `blizzard:cli-contract`; `web:client-drift`; `blizzard:compat`; `blizzard:manual-runner`.
6. **Workspaces and subscriptions as records.** Waits for piece 4. Their tables and seed; the six verbs on each; the
   `HostWorkspaces` holder every reader goes through, locks carried by id, and an empty registry; the host tick's
   rebuild of providers, samplers, and renewers; the host report from the snapshot; the workspace prompt's routes
   writing the record under the superuser check and its table deprecated; a runner naming a retired subscription running
   as though it named none; no default workspace — the `workspace-local` fallback, `init`'s scaffolded workspace, and
   the `BZ_WORKSPACE_*` variables gone, its change note marked breaking ([host-records.md](./host-records.md)
   §Workspaces and subscriptions, [runner-store.md](./runner-store.md) §No default workspace). Every test, harness
   profile, and script that leaned on the fallback or on `init`'s scaffolded workspace declares its workspace, or
   creates it through the API. Its context half: the domain's statement in `execution/hosts.md` §Bindings no longer
   halts a runner for a retired workspace or subscription; `bzh:configured-record` extends its scope to the host's
   records, whose change rows are `host_setting_changes`. Proven by `blizzard:unit-test` over each verb's refusals and
   the idempotent retire and enable; `blizzard:component-test` over a workspace repointed between two acquisitions with
   both taking the one lock, a held environment released through the rebuilt provider, every reader seeing a created
   workspace at the next tick, a host with no workspace running and claiming nothing, a retired workspace refusing its
   runner's next write until the binding is fixed while its other bindings keep claiming, and a retired subscription
   leaving its samples while its runner works on unthrottled by it; `blizzard:service-test` against the mock hub seeing
   the host report change; `blizzard:harness-check`; `blizzard:store-contract`; `blizzard:cli-contract`;
   `web:client-drift`; `blizzard:compat`.
7. **Apply and export.** Waits for pieces 5 and 6. The runner's apply and export, and `foundation/toml_emit.py`
   ([runner-api.md](./runner-api.md) §Apply). Its context half: `bzh:config-apply` extends its scope to the runner's
   document. Proven by `blizzard:unit-test` over the plan never retiring an omitted record and refusing an unknown
   runner; `blizzard:component-test` over a refused entry rolling back the whole document, a dry run reaching a
   validation refusal, and an export applied back writing nothing; `blizzard:cli-contract`; `web:client-drift`;
   `blizzard:compat`.
8. **The local page.** Waits for piece 7. The Settings tab ([runner-api.md](./runner-api.md) §The local page). Proven by
   `web:unit-test`, `web:lint`, `web:typecheck`, `web:story-render`, `web:client-drift`, `blizzard:ui` over editing a
   budget and reading `in_force` move after a step, a guest seeing no edit control, and `web:manual-panel`.
9. **The hub's settings.** Waits for piece 3. The hub-wide tables, `SETTINGS_ADMIN`, the seed, the reader, every holder
   reading per pass or per request, the routes and CLI, apply and export ([hub.md](./hub.md)). The operator pages on hub
   configuration and on config documents; the upgrade note that the file now seeds these keys. Its context half:
   `bzh:configured-record` gains the hub-wide record; `bzh:config-read-on-use` binds the hub's entries and narrows its
   scope sentence on process settings to the `boot` keys; `bzh:settings-snapshot-per-pass` extends its scope to the hub;
   `bzh:config-key-declared` gains the hub's half of its `ConfigView` sentence. Proven by `blizzard:unit-test` over the
   permission held by no tenant role; `blizzard:component-test` over a sweep's interval write reaching the next pass, a
   transcript cap write reaching the next ingest in every tenant, and the throttle rebuilt; `blizzard:service-test` on
   the controlled clock writing a sweep interval and stepping; `blizzard:store-contract`; `blizzard:cli-contract`;
   `blizzard:compat`; `web:client-drift`; `blizzard:manual-hub`.
10. **The hub board.** Waits for piece 9. The Hub settings page ([hub.md](./hub.md) §The board). Proven by
    `web:unit-test`, `web:lint`, `web:typecheck`, `web:story-render`, `web:client-drift`, and `blizzard:ui` over editing
    a sweep interval as a hub administrator and a tenant superuser finding no page.

The epic is delivered when pieces 8 and 10 have both landed. Its follow-up removals are standalone `(BREAK)` issues,
belonging to no epic, waiting until an operator decides a release may carry them: `BZ_RUNNER_TICK_SECONDS`, deprecated
by piece 2; the hub's `work_source[]` key, deprecated by piece 2; `.env`'s runner tokens, `[hub_tokens]`, and
`runners/<name>/`, deprecated by piece 5; and the `workspace_prompt` table with its seed path, deprecated by piece 6.
The default workspace is not among them: piece 6 removes it outright.

## Hub and runners on different versions

No piece crosses the fleet wire, so a host carrying any of these pieces works against any hub in its window, and a hub
carrying them against any runner in it. A runner declares its settings at registration as it always has.

## What an operator does once

Each piece's revisions are applied with `blizzard runner migrate` or `blizzard hub migrate` (`bzh:manual-migrations`).
The first start after seeds the store from the file, and from then on the file is read only for what the store lacks: an
operator who edits a seeded key in the file and restarts sees nothing change, and changes it through the API instead. An
operator who shortens the tick with `BZ_RUNNER_TICK_SECONDS` moves it to `tick_seconds` in the file before the variable
is removed.

Walking a host or a hub back to a release that predates a piece returns it to reading its file for every key that piece
made an entry: every change made through the API since is left behind in the store, unread, and the daemon runs as its
file says. That is accepted; an operator who means to walk back exports first and copies what they want into the file.

> **Warning — a walk-back strands runners added since the upgrade.** From piece 5, a runner added through the API keeps
> its hub token only in the store, sealed; nothing writes it to `.env` or `[hub_tokens]`. A host walked back to a
> release before piece 5 reads its tokens only from those places, so every runner added after the upgrade cannot
> authenticate, and the only recovery is to retire it from the hub and add it again. The owner has accepted this for now
> and does not like it: before piece 5 is filed, decide whether the store's tokens are also written somewhere the old
> release reads, or whether the walk-back is refused while such a runner exists.

## Left for later

The settings revision a session ran under stays on the runner; recording it as a fact on the hub, which comparing runs
across a fleet would want, is left for `epic:config/sweep` to ask for. Until a runner takes its side of the settings it
shares with the hub — the hub's offline cutoff, its usage staleness, and its transcript record cap — from the hub's own
settings, the two can fall out of step, and nothing checks them; that is left for later. A `boot` key changed without a
restart, a workspace whose disk blizzard reconciles, and a pair inside one daemon whose sides are both still constants
all wait for a reason to build them. Narrowing the human-lane writes that predate this epic to a role is the runner
auth's, not this epic's. Whether a runner needs its own thread group, and whether a runner's id should keep scoping the
store the way a tenant's does rather than being an ordinary entity's key, are questions this epic leaves standing for a
later one.
