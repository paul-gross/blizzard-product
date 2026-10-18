---
initiative: test-suite
refinement: scaffolded
epics:
  - foundation
  - hub-tier
  - runner-tier
  - optimization
---

# Plan — `initiative:test-suite`

Someday a daemon will be rewritten, in Python or in something else, and whoever does it will want a suite that tells
them whether the new one still keeps the old one's promises. The service tier is where that suite should live, and today
it cannot serve. A little over a third of its tests describe the hub from the outside, over HTTP. The rest run the
runner inside the test process, step its loop by calling it, and read its store through the ORM, so they can only ever
hold the current Python to its promises. Each test also mints a fixture world, starts a forge, and initializes a hub
through every migration, only to make a handful of requests and throw the whole stack away.

This initiative moves the service tier to Playwright Test, written in TypeScript, reaching blizzard only through what it
exposes: its configuration files, its HTTP API and event stream, its CLI, and the mock fleet's levers. Tests that can
share run on a few long-lived hubs per configuration, each with a mock forge and IdP of its own and shared by a handful
of test workers, since one hub is one process and serves one core's worth of tests. Each test claims a tenant of its
own, creates only what it needs inside it, and asserts only what it can see from there. `initiative:multi-tenancy` built
the boundary that makes the sharing possible, and the test suite is the first tenant it was built for.

Independence is the purpose; speed is a guard. The tier never becomes the gate's slowest job, and each epic records the
tier's CI time when it lands. What a pull request waits on today is the unit and component shards, which the
optimization epic owns, so a faster service tier alone would not shorten that wait.

## Decisions

**Playwright Test is the tier's framework.** Its fixtures are the draw: scoped, injected by name, built only when a test
asks for them, and composed from one another, so a test that names a tenant gets the hub beneath it without arranging
either. Its projects are the natural shape for hub configuration profiles. Writing the tier in another language enforces
the black-box line by construction, since a TypeScript test cannot import `blizzard.*`. Playwright runs TypeScript
without a build, and the gate checks its types. The alternatives were weighed but not spiked:

- **pytest in a package sealed from `blizzard`** has the same fixture model and would keep most test bodies, but leaves
  the boundary to packaging rather than language.
- **Go**, with the standard library or Ginkgo, orchestrates processes and reads streams well but has no fixture model of
  comparable depth.
- **Rust**'s nextest runs each test in its own process, which fights a hub shared across tests.
- **Hurl and Venom** cannot hold an event stream open or own a daemon's lifecycle.

A spike against a real hub retired the hub side's risks and found one ceiling, a single hub's one core; the foundation
plan carries what it proved and what it asks of the product. The spike never reached the runner, so the runner tier
opens with one of its own.

**The service tier stays one method with two runners.** `blizzard:service-test` keeps its name and its required CI
check. It runs the Playwright project, and beside it the pytest cases whose subject is Python itself: an adapter driven
against a mock fleet binary, which `bzh:external-cli-fake-is-service-tier` keeps in this tier because only its job
checks out the mock fleet. From the day the foundation lands, a new daemon test is written in Playwright.

**Each test moves once.** Porting a test and moving it onto the shared hub are one step: it lands in Playwright, in its
own tenant, and its Python original is deleted in the same change. Any helper another tier imports from the deleted file
moves to a support module first. The port keeps the original's function name as its title, so the harness citations that
name it move mechanically. A test whose subject is the whole hub moves to a dedicated hub rather than being weakened to
fit a shared one.

**Public surfaces only.** A test on a shared hub seeds and asserts only through HTTP, the CLI, and the mock fleet's
endpoints. It never writes or reads the hub's store, because a row the daemon did not write skips every cache, event,
and derivation the real path runs, and the test then proves a state the hub can never reach on its own. Test data is
built by the test's own fixtures through those surfaces, never by `blizzard-mock-data`, which serves agents and people
testing by hand. Where a test needs state no public surface can produce, the order of preference is the real flow, then
a surface an operator would want too, then a test-only route behind a configuration flag. Store access is allowed only
on a dedicated hub whose subject is the store itself, against files before the daemon starts or after it stops.

**The harness travels with the tests.** blizzard-context is what a fleet worker reads before writing a test, so it moves
in step: the foundation redefines the tier there, and every port re-points the citations of the tests it moves in the
same change. The daemons a Playwright run starts keep every invariant the pytest suite gives them today, the statement
guards first among them.

**The black-box line wins.** Where speed and implementation-independence pull apart, independence wins, so a shortcut
such as starting a hub from a pre-migrated database never reaches the tier.

**Nothing is optimized on its way out.** Work on code the initiative is about to delete is wasted; the fixed SSE waits,
for one, are replaced as the tests that hold them are ported, not before.

## The order

The [foundation](./foundation.md) lands first: the Playwright project, the shared-hub harness, the mock fleet changes a
shared hub needs, the blizzard-context changes that redefine the tier, and the three analytics test files ported to
prove it. The [hub tier](./hub-tier.md) follows, porting the tests that already speak HTTP and building the few surfaces
they still lack. The [runner tier](./runner-tier.md) comes last because it needs real product work: a host whose loop
can be stepped from outside, and reads of the state its tests check today through the ORM. Tests that start a real
runner host move with the runner tier even where their subject is the hub. Each epic builds the public surfaces its own
tests need, so the architecture work arrives just in time rather than in advance.

[Optimization](./optimization.md) is listed last but runs alongside the others. Its work speeds the tiers that stay in
Python and the pipeline around them, and none of it waits on the service tier's move.

## Boundaries

The initiative covers the service tier and the gate's speed. Decoupling e2e, the crash sweep, and the journey rehearsal
from the Python is `epic:test-architecture`'s, which builds on the surfaces the hub and runner tiers publish; whether
`blizzard-mock-data` seeds those tiers is decided there. Unit and component tests stay in Python and stay coupled to it
on purpose, and a service test whose subject turns out to be internal machinery moves down to the component tier rather
than being ported.
