---
epic: test-suite/foundation
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/foundation`

Before a single test can move, the tier needs somewhere to land: a Playwright project that brings up the mock fleet and
a hub once, hands each test a tenant of its own, and tears nothing down that another test still stands on. This epic
builds that ground and proves it with the first few tests to cross over.

## The Playwright project

A TypeScript Playwright Test project in the `blizzard` repo, run by a `mise` task and by a job in `upper-tiers.yml` with
the three-repo checkout the service tier already needs, since the mock fleet's binaries come from the sibling
`blizzard-mock`. It needs no browser. It runs the daemons as the installed binaries an operator runs, never as imported
code. The spike settled its toolchain:

- **Playwright Test** with no browsers installed, its fixtures carrying the hubs, the forge, and the tenants, and its
  workers in place of xdist.
- **A small streaming reader on Node's own `fetch`** for the event stream, since Playwright's request context reads a
  response whole.
- **A TOML library** to write profiles, with every key checked against `contracts/config/hub.json` until the hub refuses
  unknown keys itself.
- **`tsc --noEmit` in the gate**, because Playwright runs TypeScript without checking its types.
- **A setup project with dependencies** rather than a global setup, so a run that selects only dedicated-hub tests
  starts no shared hub.
- **A pid-file sweep** of the daemons a run starts, so a test runner killed outright leaves nothing behind.
- **One request context per hub**, because cookies ignore the port and two hubs on one host would share a session.

## The shared hub

- **Lifetime.** A few hubs per configuration profile, each started once and shared by a handful of workers, in the ratio
  [the spike's ceiling](#the-ceiling-it-found) asks this epic to measure. The mock forge is shared by them all.
- **Profiles.** A profile is a hub configuration written as TOML, never built from Python objects, which makes the hub's
  configuration format a documented contract. Each profile is a Playwright project, so a test declares the profile it
  needs by where it lives.
- **Tenant per test.** A test-scoped fixture creates a tenant through `POST /api/admin/tenants` and hands the test a
  client that names it in `X-Blizzard-Tenant` on every request. The tenant arrives holding the packaged graphs and a
  `default` project; the test adds its own runner credentials, work sources, and repositories. Once more than one tenant
  is open, the hub answers a request that names none with `409 tenant_required`, so a forgotten header fails loudly
  instead of leaking.
- **Repositories per test.** On the shared forge, each test mints repositories under names of its own. Today every test
  addresses the one fixture repository, `blizzard/<REPO_NAME>`, which only works because no two tests share a forge.
- **Teardown.** The fixture deletes its tenant afterwards, but no test's correctness depends on it: a leftover tenant is
  invisible to others by construction.
- **Dedicated hubs.** A worker-scoped fixture starts a hub of a test's own for what cannot share: a hub-wide setting no
  profile carries, a test that stops, kills, restarts, or migrates the hub, and tenant administration itself.

## Store access

A test on a shared hub seeds and asserts only through HTTP, the CLI, and the mock fleet's endpoints. Store access is
allowed only on a dedicated hub whose subject is the store, against files before the daemon starts. `blizzard-mock-data`
writes live into a running hub's SQLite, which suits demos and manual testing on a dedicated hub; it never seeds a
shared one.

## Leak guard

- **Canary.** A test in its own tenant asserts that no event, row, queue entry, or graph from another tenant reaches it
  while the rest of the suite runs.
- **Cross-check.** The shared suite runs in random order at high parallelism, and is compared from time to time with a
  run of the same tests on dedicated hubs.

## Measurements owed

`initiative:multi-tenancy` left two numbers to this epic. All tenants on a SQLite hub share one writer, so the epic
measures how many concurrent tenants a hub carries before tests queue behind each other's writes, and that measurement
decides whether the shared hubs run on SQLite at all. It also measures how long a tenant the size a service test creates
takes to tear down, and decides whether that time is part of each test's budget or hygiene that runs behind the suite.

## Proof

The three `test_analytics_*_service.py` files are the cleanest hub-side tests and move first, deleted from
`tests/service/` in the same change. They exercise the harness end to end: a profile, a tenant, the forge, an ingest,
and fleet traffic under a runner token.

## What the spike proved

A throwaway spike against a real hub and mock fleet answered the choice's risks before the harness is built on it. It
ran twenty tests on four workers, green in under a minute, with no Python and no store access anywhere in the suite.

- **Daemon lifecycles hold.** A global setup mints the fixture world, starts the forge, initializes and hosts the hub in
  about ten seconds, and stops both by pid when the run ends. A worker-scoped dedicated hub costs about the same and is
  reused across that worker's tests.
- **Tenants isolate.** On one hub, under eight concurrent workers, every tenant saw exactly its own chunks, another
  tenant's were `404`, a request naming no tenant answered `409 tenant_required`, and a runner token naming a foreign
  tenant answered `400`. No run produced a locked database, a server error, or a warning in the hub's log. Every tenant
  deleted inline, about 230 rows each, in well under a second without contention.
- **The event stream reads without a browser.** Playwright's request context reads a response whole, so a reader on
  Node's own streaming `fetch` holds the stream open. A frame arrived within milliseconds of its act, another tenant's
  subscriber received nothing, and `Last-Event-ID` resumed from its cursor. Two habits follow: event ids count per
  tenant from one, and a frame can arrive before the request that caused it returns, so a test notes its position before
  it acts.
- **The CLI already names a tenant.** Tenant-scoped `blizzard hub` verbs take `--tenant`, and without it a caller who
  can reach several tenants is refused with the list. Only an environment variable for it is missing.
- **A sign-in round trip needs nothing special.** On a dedicated hub with authentication on and a superuser declared,
  the request context followed authorize through the stub IdP and back, kept the session cookie, and acted as the hub's
  admin. Cookies ignore the port, so each hub gets a request context of its own.
- **Configuration is only partly a contract.** TypeScript edits the TOML `blizzard-hub init` writes and checks every key
  against `contracts/config/hub.json`, and the hub refuses a bad value at boot. But a misspelled key boots silently, and
  the hub never shows which value it is running with.

### The ceiling it found

One shared hub is one Python process, and it caps the suite at one core: sixteen tests took the same 33 seconds on one
worker as on four. Creating a tenant dominates, because each one mints the six packaged graphs, and at eight workers the
slowest calls approach the store's five-second busy timeout. The suite therefore runs several shared hubs per profile,
each serving a few workers, and this epic measures the hub-to-worker ratio rather than assuming one. Making a tenant
cheaper to create is a product change this epic weighs against the extra hubs.

### What it asks of the product

- **A hub that refuses an unknown configuration key** and reports the configuration it is running with, so a profile
  cannot drift silently from what a test believes it set.
- **Cheaper tenant creation**, such as minting packaged graphs lazily, if the measured ratio of hubs to workers is too
  costly without it.
- **One call to furnish a tenant** — runner, secret, work source, and repository — in place of the five a test makes
  today, if an operator would want it too.
- **An environment variable for the CLI's tenant**, beside `BZ_HUB_URL`.

Settings that stay hub-wide — route-token and produces enforcement, authentication, sweep intervals, forge write closure
— ride profiles or dedicated hubs; none of them becomes per-tenant for the sake of the tests.
