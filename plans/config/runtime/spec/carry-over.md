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
wire, so none extends `blizzard-mock`. The runner half is pieces 1 to 7; the hub half, 8 and 9, waits only on piece 1
and may land beside the runner half.

1. **Lifetimes in the registry.** Waits for nothing. `Lifetime` and `ConfigKey.lifetime` and `env`, every key of both
   registries given its lifetime from [settings.md](./settings.md), the variables that supply `boot` keys declared, both
   contract documents gaining the two fields ([lifetimes.md](./lifetimes.md)). The host file's `tick_seconds` key,
   `pass`, read at boot for now; `BZ_RUNNER_TICK_SECONDS` deprecated, honored only when the file states none; both
   drivers taking an interval callable. Every test and script that sets the variable moves to the file key in the same
   change: the crash suite's support (`tests/crash/support.py:59`), the runner and journey service specs, the two
   store-migration service tests, the rehearsal installer, the runner-host harness check, the step-pass component test,
   and the OpenCode bundle probe. Its context half: `bzh:config-key-declared` names the lifetime and the `env` rule.
   Proven by `blizzard:unit-test` over a registry refusing a key with no lifetime and a non-`boot` key with `env`;
   `blizzard:config-contract`; `blizzard:compat`; `blizzard:harness-check`; `blizzard:crash-sweep` on the file key.
2. **A runner's settings, live.** Waits for piece 1. The runner-scoped settings tables and their revision, the seed of
   each runner's keys, the reader, the tick's evaluation and the context's rebuild, the spawn's revisions, the superuser
   check, the runner's settings routes and their `settings` CLI verbs without `--host`, `apply`, or `export`
   ([runner-store.md](./runner-store.md), [runner-api.md](./runner-api.md), [passes.md](./passes.md) §A runner's tick,
   §Sessions). Each per-runner key becomes an entry in this piece, so no key is ever ignored in the file without a way
   to change it. The operator page on runner configuration gains entries, lifetimes, and the seed; its upgrade note says
   the file now seeds a runner's keys and is not read for them again. Its context half:
   `bzh:settings-snapshot-per-pass`; `bzh:config-read-on-use` binding entries; the domain's binding statement in
   `execution/hosts.md` §Bindings no longer halting a runner for a retired workspace. Proven by `blizzard:unit-test`
   over every refusal of a write and over the seed's three cases, an unset entry surviving a restart against a file
   stating the key; `blizzard:component-test` over a write landing between two ticks and taking hold at the second, a
   write during a tick not reaching it, a refused write leaving the runner as it was, two writes racing one revision,
   lowering `agents` below the running sessions stopping claims and no session, and a spawn recording its revisions;
   `blizzard:service-test` on a stepped host writing `throttle.budget.usd` and stepping the tick;
   `blizzard:store-contract`; `blizzard:cli-contract`; `blizzard:compat`; `blizzard:rehearsal`, an upgraded host seeded
   from its file running exactly as before; and `blizzard:manual-runner`.
3. **The host's settings, live.** Waits for piece 2. The host-level tables, their seed, the host tick's and trace
   sweep's evaluation, `tick_seconds` and `tracing.sweep_seconds` read live with the wake on write, `--host` on the CLI
   ([passes.md](./passes.md) §The trace sweep, §The loop's interval). Proven by `blizzard:unit-test` over the driver's
   wake running at once or waiting out the rest; `blizzard:component-test` over a `tick_seconds` write from an hour to a
   second taking hold without waiting the hour, and a tracing write reaching the next sweep; `blizzard:harness-check`;
   `blizzard:store-contract`; `blizzard:cli-contract`; `blizzard:compat`.
4. **Runners on a running host.** Waits for piece 3. The hosted-runner tables and their seed, the add and retire routes,
   the host tick's reconcile, a runner's drain alone, `RetiredRunnerWatch` writing its fact, the app's settings read
   from the host rather than its first runner, `runner add` and `retire` reaching a running host
   ([host-records.md](./host-records.md) §Runners). Its context half: `bzh:crash-exemptions-runner` records the add's
   window. The operator page on adding runners no longer says a runner waits for the host's next start. Proven by
   `blizzard:component-test` over an add starting the runner at the next tick and leaving its neighbours' ticks
   unbroken, a forced retire draining one runner's sessions and no other's, the host's first runner retired with the
   page still served, and a hub-side retirement stopping the runner's threads; `blizzard:service-test` adding a runner
   to a stepped host and claiming with it; `blizzard:crash-sweep` with crash points inside the add;
   `blizzard:cli-contract`; `blizzard:compat`; `blizzard:manual-runner`.
5. **Workspaces and subscriptions as records.** Waits for piece 3. Their tables and seed, the six verbs on each, the
   host tick's rebuild of providers, samplers, and renewers, the host report from the snapshot, the workspace prompt's
   routes writing the record and its table deprecated ([host-records.md](./host-records.md) §Workspaces and
   subscriptions). Proven by `blizzard:unit-test` over each verb's refusals and the idempotent retire and enable;
   `blizzard:component-test` over a workspace repointed between two acquisitions, a held environment released through
   the rebuilt provider, a retired workspace leaving its runner's other bindings claiming, and a retired subscription
   leaving its samples; `blizzard:service-test` against the mock hub seeing the host report change;
   `blizzard:store-contract`; `blizzard:cli-contract`; `blizzard:compat`.
6. **Apply and export.** Waits for pieces 4 and 5. The runner's apply and export ([runner-api.md](./runner-api.md)
   §Apply). Proven by `blizzard:unit-test` over the plan never retiring an omitted record and refusing an unknown
   runner; `blizzard:component-test` over a refused entry rolling back the whole document, a dry run reaching a
   validation refusal, and an export applied back writing nothing; `blizzard:cli-contract`; `blizzard:compat`.
7. **The local page.** Waits for piece 6. The Settings tab ([runner-api.md](./runner-api.md) §The local page). Proven by
   `web:unit-test`, `web:lint`, `web:typecheck`, `web:story-render`, `web:client-drift`, `blizzard:ui` over editing a
   budget and reading `in_force` move after a step, a guest seeing no edit control, and `web:manual-panel`.
8. **The hub's settings.** Waits for piece 1. The hub-wide tables, `SETTINGS_ADMIN`, the seed, the reader, every holder
   reading per pass or per request, the routes and CLI, apply and export ([hub.md](./hub.md)). The operator pages on hub
   configuration and on config documents; the upgrade note that the file now seeds these keys. Its context half:
   `bzh:configured-record` and `bzh:config-read-on-use` gaining the hub-wide record. Proven by `blizzard:unit-test` over
   the permission held by no tenant role; `blizzard:component-test` over a sweep's interval write reaching the next
   pass, a transcript cap write reaching the next ingest in every tenant, and the throttle rebuilt;
   `blizzard:service-test` on the controlled clock writing a sweep interval and stepping; `blizzard:store-contract`;
   `blizzard:cli-contract`; `blizzard:compat`; `web:client-drift`; `blizzard:manual-hub`.
9. **The hub board.** Waits for piece 8. The Hub settings page ([hub.md](./hub.md) §The board). Proven by
   `web:unit-test`, `web:lint`, `web:typecheck`, `web:story-render`, `web:client-drift`, and `blizzard:ui` over editing
   a sweep interval as a hub administrator and a tenant superuser finding no page.

The epic is delivered when pieces 7 and 9 have both landed. Its follow-up removals are standalone `(BREAK)` issues,
belonging to no epic, waiting until an operator decides a release may carry them: `BZ_RUNNER_TICK_SECONDS`, deprecated
by piece 1; and the `workspace_prompt` table with its seed path, deprecated by piece 5.

## Hub and runners on different versions

No piece crosses the fleet wire, so a host carrying any of these pieces works against any hub in its window, and a hub
carrying them against any runner in it. A runner declares its settings at registration as it always has.

## What an operator does once

Each piece's revisions are applied with `blizzard runner migrate` or `blizzard hub migrate` (`bzh:manual-migrations`).
The first start after seeds the store from the file, and from then on the file is read only for what the store lacks: an
operator who edits a seeded key in the file and restarts sees nothing change, and changes it through the API instead. An
operator who shortens the tick with `BZ_RUNNER_TICK_SECONDS` moves it to `tick_seconds` in the file before the variable
is removed.

## Left for later

The settings revision a session ran under stays on the runner; recording it as a fact on the hub, which comparing runs
across a fleet would want, is left for `epic:config/sweep` to ask for. Until a runner takes its side of the settings it
shares with the hub — the hub's offline cutoff, its usage staleness, and its transcript record cap — from the hub's own
settings, the two can fall out of step, and nothing checks them; that is left for later. A `boot` key changed without a
restart, a workspace whose disk blizzard reconciles, and a pair inside one daemon whose sides are both still constants
all wait for a reason to build them. Narrowing the human-lane writes that predate this epic to a role is the runner
auth's, not this epic's.
