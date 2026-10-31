---
epic: test-suite/controlled-clock
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/controlled-clock`

Some of blizzard's most important promises are about time. A budget's trailing window lets spend age out, a usage floor
opens when its period resets, the forge is read afresh once a floor of quiet has passed, and an egress window closes on
a whole second. A test of any of those promises today has one way to see it kept: start the daemons and wait, in real
time, for the moment to arrive. The wait is too long, so every run pays for it, or too short, so the test fails on a
slow machine. The rulebook even makes it legitimate, asking a test whose subject is expiry to poll with a deadline wide
enough to cover the window. A promise about an hour, or a night, cannot be tested at all.

The daemons already keep time through one seam. Every instant the hub and the runner stamp or judge comes from an
injected clock, and the component tier already moves that clock by hand. What the service and UI tiers lack is a way to
reach it from outside, through the same public surfaces the initiative holds them to. This epic gives the hub and the
runner host a clock that stands still until a test moves it, and an API to move it: set it to an instant, or advance it
by a duration. The answer arrives once the change has been applied, and the test then steps a sweep or a host pass and
reads what happened. The test no longer waits for a window to pass. It moves the clock past the window and checks the
result.

Stepping is what makes this safe across several processes. The harness already runs every hub sweep and every runner
host stepped, so no pass runs unless a test asks for one. A test that moves the clock on the hub, then the runner host,
then the mock forge, does so with nothing running in between that could read one process's time and act on another's.
Every pass the test then runs sees a single timeline.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## Decisions

**Only domain time is controlled.** The controlled clock replaces the instants the product reasons about: leases,
floors, windows, sessions, and stamps. Mechanical deadlines stay on real time. These include a subprocess's timeout, an
HTTP client's timeout, a retry's backoff within one request, and the wait for a daemon to become ready. Each exists to
catch a hang, and a hang happens in real time. Span timestamps stay real too, since telemetry only observes.

**It is a test fence, not a feature.** An operator gets nothing from a frozen clock, so the mode is no configuration key
and its routes appear in no published contract. It is switched on by an environment variable and honoured only behind a
fence variable of its own, the way an armed crash point fires only when its fence is set. A supported release never
ships it as a surface, so it stays outside every compatibility promise.

**The clock starts at the real time of boot.** A daemon in the controlled mode reads the machine's clock once, as it
starts, and stands still from there. The board's relative ages stay sensible, and a stamp written during boot is an
ordinary present-day instant. The harness then aligns every daemon in a test's stack to one instant before the test
begins, so processes started seconds apart share a single timeline.

**Time only moves forward.** Setting the clock to an instant earlier than its reading is refused. A real clock that
steps backwards is a failure blizzard does not set out to survive, and a test that wants an earlier world starts a new
stack.

**A test that moves time owns the clock it moves.** There is one clock per hub, not one per tenant, so moving it on a
shared hub would expire leases and sessions in every neighbouring test. A test that controls the clock runs at
`fresh-hub`, or on a held stack shared only by its own block. A runner test that already owns its mock hub and host owns
its clock with them.

**Every spec reads time from the harness.** A spec that needs an instant asks the clock fixture for it, whether the
stack runs controlled or not. A check refuses any spec that reads the machine's clock, so a test can no longer race the
clock without saying so in the spec.

## What it moves

Three specs wait out real windows today, and they move first: the budget lever's thirty-second trailing window, the
forge-status sweep's full-pass floor, and the egress night's fifteen-second slow station and its whole-second window
close. Three more read the machine's clock to build instants the daemons then judge against their own: the subscription
floor, release, and utilization-lever specs. Under a controlled clock, they can also prove the moment a floor opens at
its reset, which none of them can reach today. [spec/ports.md](./spec/ports.md) carries the roster.

## Boundaries

The crash sweep, the pytest half of the service tier, and the component tier are untouched: the first kills real
processes on purpose, and the other two already hold their clocks in-process. The OpenTelemetry collector's own sampling
wait is a third party's real time. A worker's own transcript, produced by a harness process outside the daemons, stays
on the machine's clock unless the spec's open question settles otherwise.

The hub's flow driver is the one hub lane with a timer of its own even when every sweep is stepped. Its floor reads the
controlled clock, so an advance can let it act on its next wake without being asked. Tests wait on its effects as they
do today, and giving it a stepped mode is not this epic's work.

## Done

The hub, the runner host, and the mock fleet's daemons run on a controlled clock when the fence allows it. A test moves
every clock in its stack, and the board's along with them, through one fixture. No spec waits out a real window or reads
the machine's clock without stating why. The rulebook's exception for waiting on expiry is gone.
