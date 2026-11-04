# The clock fixture

## Asking for it

A test asks for a controlled stack with `test.use({ clockMode: 'controlled' })`. The default is `'system'`. The option
is not named `clock`, since `test.use` with a fixture's own name replaces the fixture. The `clock` fixture is built for
every test either way, so a spec reads time through one surface whatever its stack runs on:

| Member             | On a `'controlled'` stack                                                               | On a `'system'` stack             |
| ------------------ | --------------------------------------------------------------------------------------- | --------------------------------- |
| `now()`            | The stack's instant, as the harness last set it                                         | The machine's time                |
| `advance(seconds)` | Moves every clock in the stack, then the browser's, and resolves once all have answered | Refused, naming the option to set |
| `set(instant)`     | The same, to an instant                                                                 | Refused                           |

`now()` on a controlled stack is the harness's own record of the instant it last set. It is not a fresh read of a
daemon, because once the stack is aligned every daemon reads exactly that instant.

## Who may ask

A controlled clock is one clock per process, so a test controls it only where no other test shares the process:

- **A stack with a real hub** asks at `fresh-hub`, or on a held stack (`test.use({ held: … })`). A held stack aligns
  once when the first test of its block builds it, and every test of the block shares its timeline in order, as the
  egress night does.
- **A runner test in `tests/service/runner/`** owns its mock hub, its host, and its vendor counterpart already, so it
  may ask at any isolation level.
- **Anything else is refused** when the fixture is built, with a message naming `fresh-hub`, as `factories.ts` refuses
  an all-tenant act below it.

A controlled clock pairs only with stepped daemons. `checkStepped` (`tests/harness/profiles.ts`) refuses a timed sweep
only in a profile's `profile.toml`, so the fixture adds its own refusal: a controlled stack refuses a timed sweep in the
test's `hubConfig` and a timed runner host (`runnerLoop: 'timed'`). A timed pass between two moves of one stack could
read the hub's new instant and the runner's old one, which is the race stepping exists to close.

## Starting the stack

The fixture adds `BLIZZARD_CLOCK=controlled` to the environment of every daemon it starts for a controlled stack: the
hub, the runner host, and the mock hub, mock runner, and mock forge beside them. `tests/daemon-env.json` sets
`BLIZZARD_CLOCK_FENCE=1` for every harness daemon, and the stub IdP takes neither variable.

## Aligning a stack

Each daemon anchors at the moment it booted, so a hub started seconds before its runner host reads an earlier instant.
Before the test body runs, the fixture reads every daemon's clock and sets each to the latest reading. Every move is
forward, so no refusal can fire. A daemon whose clock route answers `404` failed to take the controlled mode, and the
fixture fails the test there, naming the daemon and the fence.

Specs also start daemons inside the test body (`runnerHosts.create`, `mockRunner(…)`), and a fresh hub may restart. A
daemon that joins a stack already aligned is aligned as it joins: set forward to the stack's instant if it reads behind,
or the whole stack moved forward to its reading if it reads ahead. A restarted daemon runs its boot work on its own boot
instant until it is aligned. `FreshHub.start()` no longer refuses to restart a stepped hub.

## Moving a stack

`advance` and `set` move the hub first, then the runner host, then the mock fleet, one daemon at a time, and then the
browser. The test steps nothing in between, and every daemon is stepped, so no pass runs while the stack is part-way
moved. The fixture applies one instant computed from its own record of `now()`: `advance` sends each daemon
`{ "to": … }` rather than its own `advance_seconds`. If one daemon's reading had drifted, an advance would carry the
drift forward, while a `to` corrects it.

An advance is real time passing for every credential the stack holds. When it crosses the hub's session lifetime
(`IDLE_TTL`, `src/blizzard/hub/auth/service.py`), the harness's own signed-in clients sign in again before the next
request rather than failing on a `401` the test never meant to cause. A test whose subject is a session expiring reads
it before the harness refreshes it, through a member that suspends the refresh for its next request.

## The browser

A UI test's page shows instants against the browser's own clock, through the board's `FLEET_CLOCK` and now-signal. After
every move, and once after alignment, the fixture calls `page.clock.setFixedTime(now)`. That fixes `Date.now()` and
`new Date()` at the stack's instant and leaves the page's timers running, so the board's polling and stream reconnects
behave as they do today. `page.clock.install` is not used, since it would fake those timers too.

## The flow-step control it retires

The control retires in a change of its own, after the fixture's
([carry-over.md](./carry-over.md#the-order-of-landings)). Until then, a hub permitted both the controlled clock and the
flow-step control refuses to boot, so one hub never has two controlled clocks.

The hub already carries a test-fenced flow-step control: `BLIZZARD_TEST_FLOW_STEP` behind `BLIZZARD_CRASH_FENCE` builds
the hub core on a `FixedClock` (`build_stepped_clock`, `src/blizzard/hub/composition.py`), stops the driver's timer, and
mounts `POST /api/admin/test-flow/step` and `POST /api/admin/test-flow/advance`. The harness asks for it with
`test.use({ flowStepped: true })` (`tests/harness/tenancy.ts`) and calls it through `api.flowStep()` and
`api.flowAdvance()` (`tests/harness/client/clients.ts`).

That change retires all of it but the step route:

- `clockMode: 'controlled'` replaces `flowStepped`, and the harness's refusal below `fresh-hub` moves with it.
- `clock.advance(seconds)` replaces `api.flowAdvance()`, and the `/advance` route goes.
- `api.flowStep()` stays, calling the route at its unchanged path.
- `BLIZZARD_TEST_FLOW_STEP` and `build_stepped_clock` go, and with them the hub's refusal to boot with both.

`tests/service/fleet/flow-turns.spec.ts` and `tests/harness/checks/fresh-hub.spec.ts` move onto the fixture in that
change.

## The vendor counterpart

`tests/harness/vendor-usage.ts` is a fixture of the test's own worker and keeps no clock. Its `defaultLevels()` reads
`clock.now()` instead of `Date.now()`, so a credential's windows reset relative to the stack's instant.

## The check against the machine's clock

`tests/eslint.config.mjs` already refuses `waitForTimeout`, `setTimeout`, `setInterval`, and `node:timers` in specs. It
gains `Date.now()` and a zero-argument `new Date()` in every file under `tests/service/` and `tests/ui/`, with a message
pointing at `clock.now()`. A spec whose subject is real elapsed time, such as a shutdown that must finish promptly,
disables the rule on that line with a reason the reviewer can judge. Harness code outside those folders may read the
machine's clock, as it does to time its own setup.

The check lands with the last port ([carry-over.md](./carry-over.md#the-order-of-landings)), since it fails every spec
not yet moved.
