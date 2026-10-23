# Surfaces contract

Two things are shown, to two audiences, and they never travel together. **The configured settings** — every lever's
value beside its current answer, and the floor — stay on the runner's machine and are shown only where the runner itself
shows its details: `blizzard runner status`, the runner panel's machine page, and the host page's row for that runner.
**The declining reason** — which throttles are saying no right now, and on whose account — goes everywhere the runner's
state goes: its own status and panel, and on to the hub, the board, and the hub's CLI. The hub is told nothing about how
a runner is configured, so the board is not committed to showing a setting, and registration carries none.

Both are derived when they are read or sent, from the gate's verdict ([gate.md](./gate.md) §What it leaves): neither is
stored on the runner, and no fact is written for either.

## The runner's own status

`RunnerStatusView` (`wire/runner_status.py:114`), which `GET /api/runner` serves and the panel's dashboard nests
(`DashboardView.runner`, `:286`), gains `throttle: ThrottleStatusView | None` — `None` only from a runner with no loop,
as `identity` is (`:133`). Its members:

| Field            | Holds                                                                                                                                                                                                                                                                   |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `state`          | `braked` while a breaker holds the runner, else `declining` while any throttle says no, else `admitting`                                                                                                                                                                |
| `floor`          | Always present: the subscription it reads (`null` for a runner that names none), whether it admits, the hold that stands, its expected reset, whether it awaits a fresh sample, when the grace opens it unconfirmed, and the hand clear that holds it open, if one does |
| `levers`         | One entry per configured lever, in the gate's order: its name, its setting in operator words, whether it admits, the reading it judged, and `until`                                                                                                                     |
| `on_decline`     | `hold` or `release`, for a chunk the gate parks between steps ([release.md](./release.md))                                                                                                                                                                              |
| `on_usage_limit` | `hold` or `release`, for a lease a usage limit parks mid-step ([release.md](./release.md) §Releasing a usage-limited lease)                                                                                                                                             |

A lever the runner does not configure has no entry; the floor is never absent. While a breaker holds the runner, `state`
says so and every answer is still shown, so an operator about to clear a breaker can see whether a throttle would hold
the runner back the moment they do. `RunnerStatusService.summary` (`runner/status/view.py:295`) builds the view beside
the `PauseState` it already reads (`:296`–`:306`), asking `ThrottleGate` through its read seams alone, and the host
page's `HostRunnerView` (`wire/runner_host.py:36`) gains the same `throttle` beside its `pause`. Both are routes of the
runner's own API, so `openapi/runner.openapi.json` and the panel's generated client (`web:client-drift`) pick them up,
and neither reaches the hub.

## `blizzard runner status`

The status verb prints its brake line from the same read (`runner/cli/control.py:163`–`167`). That line is unchanged,
and a `throttle` block follows it — the state and, when declining, the throttles saying no; then one line for the floor
and one per configured lever, each with its setting, its reading, and its answer; then the two hold-or-release choices,
side by side:

```text
runner r-claude (rn_01M3MY1NGQR8WE1ZBRKNREFNE4)  workspace=blizzard-claude
  running
  throttle: declining — floor, pace
    floor        anthropic: exhausted — usage limit via claude_code, resets 14:00Z         no
    pace         7d, lead 5 points, 1 above 90%: 86% spent, 71% elapsed                   no   until Thu 02:10Z
    utilization  below 95% on 5h, 7d: 5h 40%, 7d 86%                                       yes
    budget       $40.00 over 24h: ~$12.40                                                  yes
    schedule     mon–fri 19:00–08:00 PDT: inside                                           yes
    on_decline   hold          on_usage_limit   release
```

A pace with a taper shows each step after its base lead, as above, and its reading names the step in force once one is
(`lead 1 above 90%`, [levers.md](./levers.md) §Pacing). A runner with no `[runner.throttle]` prints the floor and both
choices at `hold` alone; a runner naming no subscription prints its own floor as `floor        this runner: open`. A
hold past its reset says which way it is waiting —
`anthropic: exhausted — reset passed, no sample since, opens at
15:00Z` while the grace runs — and a floor a hand clear
holds open says so for as long as the clear is what keeps it open: `anthropic: open — cleared by paul at 09:14Z`
([exhaustion.md](./exhaustion.md) §Clearing by hand).

Every instant the block prints is UTC with its `Z`, as the floor's reset is. A schedule's spans are the one exception:
they are the host's wall-clock time, as configured ([levers.md](./levers.md) §The schedule), so the schedule's line
names the host's zone beside them, and its `until` is UTC like every other instant. `docs/deployment/control-verbs.md`
names the block beside the `gates:` line it already describes.

## The runner's panel

The machine page (`web/projects/runner/src/app/machine/machine-page.html`) gains a fourth panel, `throttle`, beside the
pause control it ranks under. It is a self-fetching container, `LocalThrottle` in
`web/projects/runner/src/app/machine/throttle.ts`, over the runner's one dashboard read (`core/status.query.ts`),
rendering a presentational `ThrottleView` that takes the `ThrottleStatusView` as its input and injects nothing
(`bzh:frontend-container-presentational`): a fact-list row for the floor and one per lever, as `app-info-view.ts`
renders its rows, each with its setting — a pace's taper steps listed under its base lead — its reading, its yes or no,
and its `until`, and one line holding both hold-or-release choices, `on_decline` beside `on_usage_limit`. The pause
control's reason (`app-pause-control.ts:65`) keeps showing only the breaker's.

While the floor holds, its row carries a Clear action ([exhaustion.md](./exhaustion.md) §Clearing by hand). The view
asks first through `KitConfirmDialog`, as the chunk dock's pause does (`board/chunk-dock/machine-detail-header.ts:113`),
naming what the clear lifts — `Clear anthropic's floor for every runner on this host spending it?`, or this runner alone
when it names no subscription — and emits `clearFloor` once confirmed. The container runs `injectFloorClearMutation` in
`core/status.query.ts`, beside `injectLocalPauseMutation` (`:74`), against the subscription's route or the runner's as
the floor names, under a key of its own in `core/mutation-keys.ts`, and the dashboard read refetches on success. The
host page's rows carry no action: clearing is done from the runner it lifts.

The host page shows each runner's details in its row (`web/projects/runner/src/app/host/host-runners-view.html`), and
the throttle goes there too: `hostRunnerRows` (`host/host-runners.model.ts:32`) gains a `throttle` label beside its
`brake` (`brakeLabel`, `:24`) — the state and the first declining throttle with its expected reset — and one line per
configured lever with its answer, under the binding lines. Both pages word a lever through one model function,
`throttleLines` in the runner app's `core/throttle.model.ts`, so the two never describe a lever differently.

## The hub, the board, and the hub's CLI

**Registration.** `RunnerRegistrationRequest` (`wire/runner.py:73`) gains `declining: list[RunnerDeclining] | None`:
`None` from a runner that predates it, `[]` from one admitting or braked, and otherwise one member per throttle saying
no — `lever` (`floor`, `pace`, `utilization`, `budget`, `schedule`, carried as a string so a newer runner's lever does
not fail an older hub's parse), `detail` in the same words the runner's own status uses, `subscription` when the reason
belongs to one, and `until`, the expected reset or opening, when known. A braked runner sends `[]`: its brake already
reaches the hub as its own fact, and a throttle governs only a runner no breaker holds. Registration runs on every
tick's PULL (`runner/lifecycle/registration.py:132`), so the hub's reading is at most one tick old, and each
registration supersedes the last whole, as capabilities do — a runner's own assertion, never a condition the hub derives
(`domain/execution/responsibilities.md`). No setting rides it: not a lever's value, not a line.

**Storage and views.** `runner_registrations` (`hub/store/schema.py:1909`) gains a nullable text column, `declining`,
written on every registration as `capabilities` is (`:1937`). `RunnerView` and `RunnerRegistryView`
(`wire/runner.py:244`, `:283`) gain `declining: list[RunnerDecliningView]`, empty for a runner that sent none. The field
is additive in both directions (`bzh:fleet-wire-additive`): an older runner sends nothing and is shown declining
nothing, and an older hub ignores the field. `openapi/hub.openapi.json` and the board's client regenerate, and
`blizzard-mock`'s mock hub stores and serves the field, its mock runner sends it, and the wire-parity guard maps it
(`bzh:wire-change-extends-mock`).

**The board.** The runner rows (`web/projects/hub/src/app/runners/runner-rows.model.ts`) gain a declining label beside
the brake's (`localPauseHint`, `:99`) — `waiting on anthropic — exhausted, resets 14:00` for the floor,
`declining — pace: 7d 86% spent, 71% elapsed` for a lever — showing the first in the gate's order, the rest in the row's
detail. It shows only for an online runner with no brake; a braked runner shows its brake as today.

**The hub's CLI.** `RunnerLine.brake` (`hub/cli/views.py:95`) prints a declining segment where it prints a brake —
`[declining: floor — anthropic exhausted, resets 14:00Z]` — so `blizzard hub status` and `blizzard hub runner` listings
say why an idle, unbraked runner is idle.
