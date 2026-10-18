---
epic: test-suite/foundation
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/foundation`

Before a single test can move, the tier needs somewhere to land: a Playwright project that brings up hubs and their mock
fleet once, hands each test a tenant of its own, and tears nothing down that another test still stands on. It also needs
the fleet's own rulebook to say where a service test now lives, or every worker who writes one in the meantime will
write it in pytest. This epic builds that ground and proves it with the first tests to cross over.

## The Playwright project

A TypeScript Playwright Test project in the `blizzard` repo, in a folder of its own under `tests/` with its own
`package.json`, outside `web/` so the Angular workspace's structure rules do not bind it. It needs no browser. It runs
the daemons as the installed binaries an operator runs, never as imported code. The spike settled its toolchain:

- **Playwright Test** with no browsers installed, its fixtures carrying the hubs, the mock fleet, and the tenants, and
  its workers in place of xdist.
- **A small streaming reader on Node's own `fetch`** for the event stream, since Playwright's request context reads a
  response whole.
- **Wire types generated from the committed `openapi/` specs**, the way `web/` generates its client, so `tsc` catches a
  renamed field instead of a hand-restated model drifting silently.
- **A TOML library** to write profiles, with every key checked against `contracts/config/hub.json` until the hub refuses
  unknown keys itself.
- **`tsc --noEmit` and a linter in the gate**, because Playwright runs TypeScript without checking its types.
- **A setup project with dependencies** rather than a global setup, so a run that selects only dedicated-hub tests
  starts no shared hub.
- **A pid-file sweep** of the daemons a run starts, so a test runner killed outright leaves nothing behind.
- **One request context per hub**, because cookies ignore the port and two hubs on one host would share a session.

`mise run service-test` runs the Playwright project and the pytest cases that stay in the tier, and both report under
the required check `blizzard:service-test` already has, in `upper-tiers.yml` and in `release.yml`. Both jobs keep the
two-repo checkout of `blizzard` and `blizzard-mock` and gain Node. `test_winter_free_ci.py` learns to recognise the
Playwright job as one that runs tests, and to scan its TypeScript.

## The daemons' environment

Every hub and runner the project starts, shared or dedicated, runs under the same environment the pytest suite gives the
daemons it spawns today: both statement guards on, strict configuration reads, commit signing off, the legacy forge
variables removed, and inherited tracing variables stripped. The CLI never reads the developer's own session file. That
contract lives in one place both suites read, so the two cannot drift, and a guard refusal in any daemon's log fails the
run. On a shared hub the tenant guard matters more than anywhere else, because it is what catches a cross-tenant
statement inside the daemon before any test could see its effect.

## The shared hub

- **Lifetime.** A few hubs per configuration profile, each started once and shared by a handful of workers, in the ratio
  [the spike's ceiling](#the-ceiling-it-found) asks this epic to measure. A worker reaches its hub by its index.
- **A mock fleet per hub.** Each shared hub gets a mock forge and a stub IdP of its own, started beside it. Neither runs
  migrations, so the cost is small, and it keeps one hub's tests from racing another's on mock state.
- **Profiles.** A profile is a hub configuration written as TOML, never built from Python objects, which makes the hub's
  configuration format a documented contract. Each profile is a Playwright project, so a test declares the profile it
  needs by where it lives.
- **Tenant per test.** A test-scoped fixture creates a tenant through `POST /api/admin/tenants` and hands the test a
  client that names it in `X-Blizzard-Tenant` on every request. The tenant arrives holding the packaged graphs and a
  `default` project. The test furnishes the rest through the surfaces an operator uses: secrets, then
  `POST /api/config/apply` for work sources, repositories, and project links in one call, then the runners it needs.
  Once more than one tenant is open, the hub answers a request that names none with `409 tenant_required`, so a
  forgotten header fails loudly instead of leaking.
- **Repositories per test.** Each test works in repositories under names of its own. Today every test addresses the one
  fixture repository, `blizzard/<REPO_NAME>`, which only works because no two tests share a forge, and the mock forge
  cannot create a repository at runtime. This epic gives it that ability, in `blizzard-mock`.
- **Mock state stays the test's own.** A test on a shared hub touches no mock state another test on that hub reads. A
  forge lever is scoped to the test's own repository or issue, and nothing resets the forge's levers wholesale. A mock
  runner is always started with a token minted in the test's tenant, since one left to add itself names no tenant and is
  refused. The stub IdP holds one identity for the next sign-in, so it either learns to key that identity per sign-in or
  each hub on the authentication profile serves one worker.
- **Teardown.** The fixture deletes its tenant afterwards, but no test's correctness depends on it: a leftover tenant is
  invisible to others by construction.
- **Dedicated hubs.** What cannot share gets a hub of its own, in one of two shapes. A worker-scoped hub carries a
  hub-wide setting no profile does, for tests that leave it unchanged. A test-scoped fresh hub serves a test that stops,
  kills, restarts, or migrates the hub, and a test whose subject is something multi-tenancy leaves global: tenant
  administration, the hub event log, users and hub admins, the superuser bootstrap, signing keys, tracing and egress
  export, or whether a hub-wide loop runs at all. A fresh hub costs about ten seconds, so the hub tier counts them
  before it ports.

## Test data

Fixtures build every test's data through the surfaces above. `blizzard-mock-data` writes live into a running hub's
SQLite, which suits agents and people testing by hand; it never seeds a test in this tier. Store access is allowed only
on a dedicated hub whose subject is the store, against files before the daemon starts or after it stops.

## Fact kinds as data

The fact-kind vocabulary in `foundation/fact_kinds.py` is committed under `contracts/`, with a drift test and a
verification method like those for `contracts/cli/`. The first ported tests push facts over `POST /api/fleet/events`, so
they name kinds from it rather than restating strings, and the `blizzard-mock` parity guard reads the same artifact for
fact kinds instead of the Python source.

## The rulebook

blizzard-context changes in a companion that lands with this epic. It redefines `blizzard:service-test` as the
Playwright project plus the pytest cases that stay, in its command, its tier roster, and its `mise` task. It says that
new daemon tests are written in Playwright. It scopes the statement-guard rule to every daemon a test spawns rather than
to rows that run pytest, and narrows the rule that test data comes from `blizzard-mock-data` so it no longer binds this
tier. The registry check learns to attribute a Playwright spec as firmly as it attributes a pytest file, the frontend
toolchain and comment rules extend to the new folder, and the hub's pre-push sweep looks there too. A companion rule
requires every port to move its citations in the same change.

## Leak guard

- **Canary.** On every shared hub, a tenant created by the setup project holds its event stream open for the whole run
  and asserts at the end that no event, row, queue entry, or graph from another tenant ever reached it.
- **Cross-check.** The harness can run any shared test on a dedicated hub by a flag. The release workflow runs the
  shared suite both ways, in random order at high parallelism, and the two runs must pass the same tests. That is what
  catches a test that passes only because a neighbour's activity did its work for it.
- **No shared logs.** A test that asserts on a daemon's log runs on a dedicated hub, since a shared hub's log
  interleaves every tenant's lines.

## Measurements owed

`initiative:multi-tenancy` left two numbers to this epic, and the spike added a third. All three are taken on the CI
runner's shape, four vCPUs, because a ratio found on a twenty-core laptop would not hold there.

- **Hubs per worker.** How many workers a shared hub serves before its one core becomes the wait, with and without the
  statement guards' cost, which falls on that core.
- **Store contention.** How many concurrent tenants a SQLite hub carries before tests queue behind each other's writes.
  The shared hubs run on SQLite unless the slowest store waits at the chosen ratio come within reach of the five-second
  busy timeout; then they move to Postgres, which also amends the harness rule that tests run on SQLite and adds a
  database service to the CI jobs.
- **Teardown.** How long a tenant the size a service test creates takes to delete, which decides whether that time is
  part of each test's budget or hygiene that runs behind the suite.

## Proof

`test_analytics_service.py`, `test_analytics_events_service.py`, and `test_analytics_operational_service.py` are the
cleanest hub-side tests and move first, deleted from `tests/service/` in the same change. They exercise the harness end
to end: a profile, a tenant, the forge, an ingest, and fleet traffic under a runner token.

## What the spike proved

A throwaway spike against a real hub and mock fleet answered the choice's risks before the harness is built on it. It
ran twenty tests on four workers, green in under a minute, with no Python and no store access anywhere in the suite.

- **Daemon lifecycles hold.** A setup step mints the fixture world, starts the forge, initializes and hosts the hub in
  about ten seconds, and stops both by pid when the run ends. A worker-scoped dedicated hub costs about the same and is
  reused across that worker's tests.
- **Tenants isolate.** On one hub, under eight concurrent workers, every tenant saw exactly its own chunks, another
  tenant's were `404`, a request naming no tenant answered `409 tenant_required`, and a runner token naming a foreign
  tenant answered `400`. No run produced a locked database, a server error, or a warning in the hub's log. Every tenant
  deleted inline, about 230 rows each.
- **The event stream reads without a browser.** A reader on Node's own streaming `fetch` holds the stream open. A frame
  arrived within milliseconds of its act, another tenant's subscriber received nothing, and `Last-Event-ID` resumed from
  its cursor. Event ids count per tenant from one, and a frame can arrive before the request that caused it returns.
- **The CLI already names a tenant.** Tenant-scoped `blizzard hub` verbs take `--tenant`, and without it a caller who
  can reach several tenants is refused with the list. Only an environment variable for it is missing.
- **A sign-in round trip needs nothing special.** On a dedicated hub with authentication on and a superuser declared,
  the request context followed authorize through the stub IdP and back, kept the session cookie, and acted as the hub's
  admin.
- **Configuration is only partly a contract.** TypeScript edits the TOML `blizzard-hub init` writes and checks every key
  against `contracts/config/hub.json`, and the hub refuses a bad value at boot. But a misspelled key boots silently, and
  the hub never shows which value it is running with.

### The ceiling it found

One shared hub is one Python process, and it caps the tests it serves at one core: sixteen tests took the same 33
seconds on one worker as on four. Creating a tenant dominates, because each one mints the six packaged graphs, and at
eight workers the slowest calls approach the store's five-second busy timeout. The spike's tests were back-to-back HTTP;
real tests spend much of their time waiting on the system, so a hub will carry more than one worker, and this epic
measures how many rather than assuming.

### What it asks of the product

This epic builds the first and last of these. The middle one is built here only if the measured ratio asks for more hubs
than the CI runner has cores.

- **A hub that refuses an unknown configuration key** and reports the configuration it is running with, so a profile
  cannot drift silently from what a test believes it set.
- **Cheaper tenant creation**, such as minting packaged graphs lazily, which an operator creating tenants would feel
  too.
- **An environment variable for the CLI's tenant**, beside `BZ_HUB_URL`.

Settings that stay hub-wide — route-token and produces enforcement, authentication, sweep intervals, forge write closure
— ride profiles or dedicated hubs; none of them becomes per-tenant for the sake of the tests.

## Done

The three analytics files run in the Playwright project under `blizzard:service-test`'s required check, in both
workflows, and their Python originals and blizzard-context citations are gone. Every daemon the project starts runs
under the shared environment contract. The mock forge creates repositories on demand, the canary runs on every shared
hub, the cross-check runs in the release workflow, and the fact-kind contract is committed. The three measurements are
recorded with the decisions they drove, and the rulebook companion has landed.
