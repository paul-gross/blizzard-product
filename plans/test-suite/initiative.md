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

Every chunk the fleet lands waits on the gate, and the service tier is where that wait is least deserved. Each of its
tests mints a fixture world, starts a forge, initializes a hub through every migration, and hosts it, only to make a
handful of requests and throw the whole stack away. The tests are also less independent than they look. About half
describe the hub from the outside, over HTTP. The rest run the runner inside the test process, step its loop by calling
it, and read its store through the ORM, so they can only ever hold the current Python to its promises.

This initiative moves the service tier onto a few long-lived hubs per configuration, each shared by a handful of test
workers, since one hub is one process and serves one core's worth of tests. Each test claims a tenant of its own,
creates only what it needs inside it, and asserts only what it can see from there. The tests move to Playwright Test,
written in TypeScript, and reach blizzard only through what it exposes: its configuration files, its HTTP API and event
stream, its CLI, and the mock fleet's levers. `initiative:multi-tenancy` built the boundary that makes the sharing
possible, and the test suite is the first tenant it was built for.

## Decisions

**Playwright Test is the tier's framework.** Its fixtures are the draw: scoped, injected by name, built only when a test
asks for them, and composed from one another, so a test that names a tenant gets the hub beneath it without arranging
either. Its projects are the natural shape for hub configuration profiles. Writing the tier in another language enforces
the black-box line by construction, since a TypeScript test cannot import `blizzard.*`, and Playwright runs TypeScript
with no build step to maintain. The alternatives were weighed but not spiked:

- **pytest in a package sealed from `blizzard`** has the same fixture model and would keep most test bodies, but leaves
  the boundary to packaging rather than language.
- **Go**, with the standard library or Ginkgo, orchestrates processes and reads streams well but has no fixture model of
  comparable depth.
- **Rust**'s nextest runs each test in its own process, which fights a hub shared across tests.
- **Hurl and Venom** cannot hold an event stream open or own a daemon's lifecycle.

A spike against a real hub validated the choice's risks and found its one ceiling, a single hub's one core; the
foundation plan carries what it proved and what it asks of the product.

**Each test moves once.** Porting a test and moving it onto the shared hub are one step: it lands in Playwright, in its
own tenant, and its Python original is deleted in the same change. A test that cannot share moves once too, onto a
dedicated hub in the same project.

**Public surfaces only.** A test on a shared hub seeds and asserts only through HTTP, the CLI, and the mock fleet's
endpoints. It never writes or reads the hub's store, because a row the daemon did not write skips every cache, event,
and derivation the real path runs, and the test then proves a state the hub can never reach on its own. Where a test
needs state no public surface can produce, the order of preference is the real flow, then a surface an operator would
want too, then a test-only route behind a configuration flag. Store access is allowed only on a dedicated hub whose
subject is the store itself, against files before the daemon starts.

**The black-box line wins.** Where speed and implementation-independence pull apart, independence wins, so a shortcut
such as starting a hub from a pre-migrated database never reaches the tier.

**Nothing is optimized on its way out.** Work on code the initiative is about to delete is wasted; the fixed SSE waits,
for one, are replaced as the tests that hold them are ported, not before.

## The order

The [foundation](./foundation.md) lands first: the Playwright project, the shared-hub harness, and a first handful of
ported tests that prove it. The [hub tier](./hub-tier.md) follows, porting the tests that already speak HTTP and
building the few surfaces they still lack. The [runner tier](./runner-tier.md) comes last because it needs real product
work: the runner has to be driven and read from outside before its tests can leave the Python. Each epic builds the
public surfaces its own tests need, so the architecture work arrives just in time rather than in advance.

[Optimization](./optimization.md) is listed last but runs alongside the others. Its work speeds the tiers that stay in
Python and the pipeline around them, and none of it waits on the service tier's move.

## Boundaries

The initiative covers the service tier and the gate's speed. Decoupling e2e, the crash sweep, and the journey rehearsal
from the Python is `epic:test-architecture`'s, which builds on the surfaces this initiative publishes. Unit and
component tests stay in Python and stay coupled to it on purpose. Service-tier tests that start no daemon, and instead
drive a Python adapter against a mock binary, stay in pytest and are re-tiered by the hub-tier epic rather than ported.
