---
epic: test-suite/foundation
refinement: pristine
status: horizon
---

# Plan — `epic:test-suite/foundation`

Before a single test can move, the tier needs somewhere to land: a Playwright package that brings up hubs and their mock
fleet once, gives each test as much of a hub as it needs and no more, and tears nothing down that another test still
stands on. It also needs the fleet's own rulebook to say where a service test now lives, or every worker who writes one
in the meantime will write it in pytest. This epic builds that ground and proves it with the first tests to cross over.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## The Playwright package

A TypeScript Playwright Test package in the `blizzard` repo, rooted at `tests/` with its own `package.json` and one
Playwright config. Its folders are named for the kind of test they hold, never for the tool that runs them:

| Folder                  | What is real                 | What stands around it               |
| ----------------------- | ---------------------------- | ----------------------------------- |
| `tests/service/hub/`    | the hub                      | mock runners, forge, and IdP        |
| `tests/service/runner/` | a runner host                | the mock hub, forge, and harnesses  |
| `tests/service/fleet/`  | the hub and a runner host    | the mock forge, harnesses, and IdP  |
| `tests/ui/`             | the built board in a browser | a real hub, and the mocks behind it |

This epic builds the package and `service/hub/`; the hub and runner tiers fill the service folders, and
`epic:test-architecture` the UI folder. A Playwright project is a folder crossed with a hub profile: the profile's tests
live in a subfolder named for it, `tests/service/hub/auth/` running as `service-hub-auth`, and a folder's default
profile sits at its root, `tests/service/hub/` running as `service-hub`. A test declares its profile by where it lives,
and a selection such as `--project='service-*'` takes every profile at once. The fixtures that start daemons, mint
tenants, and run the mock fleet live once in the package and serve every project. The package shares `tests/` with the
Python suites until they have moved, and pytest is told not to descend into `node_modules`; the service tier's pytest
half keeps its files in `tests/service/` beside the Playwright specs. The package sits outside `web/`, so the Angular
workspace's structure rules do not bind it, and only the UI project installs a browser. It runs the daemons as the
installed binaries an operator runs, never as imported code. The spike settled its toolchain:

- **Playwright Test** with no browser in the service projects, its fixtures carrying the hubs, the mock fleet, and the
  tenants, and its workers in place of xdist.
- **A small streaming reader on Node's own `fetch`** for the event stream, since Playwright's request context reads a
  response whole.
- **Wire types generated from the committed `openapi/` specs**, the way `web/` generates its client, so `tsc` catches a
  renamed field instead of a hand-restated model drifting silently.
- **A TOML library** to write profiles, with every key checked against `contracts/config/hub.json` before any hub
  starts. The hub ignores a key it does not know, so a misspelled setting in a profile would otherwise leave every test
  on it believing in a configuration the hub never ran. The check is the harness's own and permanent; it asks nothing of
  the hub.
- **`tsc --noEmit` and a linter in the gate**, because Playwright runs TypeScript without checking its types.
- **A setup project with dependencies** rather than a global setup, so a run that selects no test in a profile starts
  none of that profile's hubs.
- **One request context per hub**, because cookies ignore the port and two hubs on one host would share a session.

## In CI

Two `mise` tasks run the tier. `mise run playwright` runs the Playwright package from `tests/` and passes anything after
`--` to `playwright test`, so one task serves a developer running one folder (`--project=service-hub`), one test
(`--grep`), or one shard (`--shard=2/2`), and later the UI project as well. `mise run service-test` runs only the pytest
half. `blizzard:service-test` is the two together, and both report under the required check it already has, in
`upper-tiers.yml` and in `release.yml`.

- **Two shards.** The Playwright service projects run as two matrix jobs, `--shard=1/2` and `--shard=2/2`, beside the
  pytest half's job, and the existing summary job passes only when every one of them did. Each shard is its own machine
  and starts its own shared hubs and common tenant, so the hubs-per-worker measurement is taken against two shards, and
  a later change to the count is weighed against it. Each shard runs two workers to start, and the count rises only as
  the measurement and GitHub's runners allow.
- **A pinned toolchain.** Node is pinned through `mise`, the package installs with `npm ci` from its committed lockfile,
  and `node_modules` is cached between runs.
- **Failures leave evidence.** A failing job uploads Playwright's report and traces and every daemon's log, so a red run
  can be read without reproducing it.

Both jobs keep the two-repo checkout of `blizzard` and `blizzard-mock`. `test_winter_free_ci.py` learns to recognise the
Playwright jobs as ones that run tests, and to scan their TypeScript.

## The daemons' environment

Every hub and runner the package starts, shared or fresh, runs under the same environment the pytest suite gives the
daemons it spawns today: both statement guards on, strict configuration reads, commit signing off, the legacy forge
variables removed, and inherited tracing variables stripped. The CLI never reads the developer's own session file. That
contract lives in one place both suites read, so the two cannot drift, and a guard refusal in any daemon's log fails the
run. On a shared hub the tenant guard matters more than anywhere else, because it is what catches a cross-tenant
statement inside the daemon before any test could see its effect.

## The shared hub

- **Lifetime.** One hub per configuration profile to start, started once and shared by up to eight workers, a ratio
  [the spike's ceiling](#the-ceiling-it-found) asks this epic to check by measurement before it settles. A worker
  reaches its hub by its index.
- **A mock fleet per hub.** Each shared hub gets a mock forge and a stub IdP of its own, started beside it. Neither runs
  migrations, so the cost is small, and it keeps one hub's tests from racing another's on mock state.
- **Profiles.** A profile is a hub configuration written as TOML, never built from Python objects, which makes the hub's
  configuration format a documented contract. Each profile is a subfolder, and with its folder a Playwright project, as
  [the package](#the-playwright-package) lays out. A profile may be small: a hub-wide setting only a few tests need gets
  a profile whose single hub starts the first time one of them runs and serves one worker. Where a test lives decides
  how its hub is configured; its isolation level decides how much of that hub it has to itself.
- **A tenant and project per test, at its isolation level.** A test names the fixtures it needs, its tenant, its
  project, its clients, and receives each at the [isolation level](#isolation-levels) it declares, with a client that
  names that tenant in `X-Blizzard-Tenant` on every request. Once more than one tenant is open, the hub answers a
  request that names none with `409 tenant_required`, so a forgotten header fails loudly instead of leaking.
- **Repositories per test.** Each test works in repositories under names of its own. Today every test addresses the one
  fixture repository, `blizzard/<REPO_NAME>`, which only works because no two tests share a forge, and the mock forge
  cannot create a repository at runtime. This epic gives it that ability, in `blizzard-mock`.
- **Mock state stays the test's own.** A test on a shared hub touches no mock state another test on that hub reads. A
  forge lever is scoped to the test's own repository or issue, and nothing resets the forge's levers wholesale. A mock
  runner is always started with a token minted in the test's tenant, since one left to add itself names no tenant and is
  refused. The stub IdP holds one identity for the next sign-in, which a shared authentication profile cannot live with;
  no test this epic ports signs in, so the hub tier settles it.
- **Nothing is cleaned up.** A test leaves behind every tenant, project, and record it made, and the hub keeps them
  until the run stops it. Nothing depends on a cleanup that never comes: a leftover tenant is invisible to others by
  construction, and a leftover project or record is invisible to a test that asserts only on what it created. Blizzard
  retires records rather than deleting them, and a project cannot yet be retired at all, so a teardown would need
  surfaces the product does not have and the tests do not need. What leftovers cost is the hub's to carry: every open
  tenant keeps its own background passes running, so a hub's late tests run beside the sweeps of every tenant its early
  ones made, and the hubs-per-worker measurement is taken over a whole run for that reason.
- **Fresh hubs.** What cannot share a hub with any other test gets one started for it alone, the `fresh-hub` isolation
  level: a test that stops, kills, restarts, or migrates the hub, and a test whose subject is something multi-tenancy
  leaves global: tenant administration, the hub event log, users and hub admins, the superuser bootstrap, signing keys,
  tracing and egress export, or whether a hub-wide loop runs at all. A fresh hub costs about ten seconds, so the hub
  tier counts them before it ports.
- **Logs.** A shared hub's log interleaves every tenant's lines, so a test on one asserts only on lines tagged with its
  own `tenant_id`. Every line a tenant-scoped request produces carries that tag today; the lines a per-tenant background
  pass writes do not yet, and the hub tier adds it. A test asserting on an untagged line, a hub-wide one, takes
  `fresh-hub`.

## Isolation levels

An isolation level says how a test shares a real hub, so it belongs to the projects that have one: `service/hub/`,
`service/fleet/`, where it governs the hub half and the runner host is always the test's own, and `ui/`, where the
browser drives a shared hub. A test in `service/runner/` has no level to declare: it owns its host and a mock hub of its
own, which nothing else reaches.

Not every test needs a world of its own. One that creates an item, finds it listed, and deletes it can run beside a
dozen others doing the same, so long as each touches only what it made; one that starts a runner and watches it claim
work cannot, because another test's runner might claim it first. Each test declares the weakest isolation it is correct
under, and the harness gives it exactly that:

| Level           | What the test gets                                                 | Cost                |
| --------------- | ------------------------------------------------------------------ | ------------------- |
| `common`        | the common tenant's `default` project                              | nothing per test    |
| `fresh-project` | a fresh project in the common tenant                               | one request         |
| `fresh-tenant`  | a fresh tenant, its packaged graphs, and its own `default` project | a tenant's creation |
| `fresh-hub`     | a hub of its own, started for this test alone                      | about ten seconds   |

The **common tenant** is created once per shared hub by the setup project. It is not the tenant a single-tenant hub
answers header-less requests with; that is a different thing, and the name keeps them apart.

**The level is a Playwright option.** A file or a `describe` block sets it with `test.use({ isolation })`, and the
fixtures a test names read it: `tenant` and `project` build or hand over what the level gives, and the clients act
there. The level says where a test runs; the fixtures it names say what it needs there, so no test scaffolds its own
tenant, project, or person. A test at `fresh-tenant` furnishes the rest through the surfaces an operator uses: secrets,
then `POST /api/config/apply` for work sources, repositories, and project links in one call, then the runners it needs.

**What a test writes decides the level it needs.** Nothing checks a call against the level before it is sent: a check
would test nothing about blizzard, and would tax every new route with a classification, before anyone knows interference
is a problem at all. The level is the author's to choose, by what the test writes:

- **`fresh-hub`** for the hub-level routes `test_route_classification.py` already sets apart: tenant administration,
  users and hub admins, signing keys.
- **`fresh-tenant`** for writes that reach the whole tenant: secrets, `config/apply`, runners and their enrollment,
  graphs, work sources, and repositories. `config/apply` leaves unnamed records alone, so it never removes another
  test's, but what it creates is tenant-wide, and it starts here.
- **`fresh-project`** for creating a project, and for writes inside a project the test made: its routines, scopes, and
  repository and source links.
- **`common`** for writes to items and chunks the test made itself.

**A write to what the test made is the test's own.** A test that invites a person only it knows, signs them in, changes
their role, and removes them touches nothing another test can see, so it runs at `common`; the same role change aimed at
the admin the fixtures act as, or at a tenant-wide setting such as which providers may sign in, needs `fresh-tenant`.
Starting a mock runner or setting a forge lever needs `fresh-tenant` too. A test that writes beyond its level shows up
in [the cross-check](#cross-check), which is the tier's alarm for interference; if the alarm rings often, checking calls
before they are sent is the tool to reach for then.

**Reads are a rule too.** A call to list chunks is fine at any level; asserting the list holds exactly three is not, and
that lives in the assertion. At `common` and `fresh-project` the client makes the right thing the easy one: a list comes
back already narrowed to the ids the test created, and the whole list takes an explicit `api.unfiltered` call that a
reviewer will see. Beyond that, a test at those two levels names everything it creates uniquely, asserts no count,
emptiness, or absence across the tenant, filters its event stream to its own ids, chooses for a negative assertion's
barrier an act only it would emit, and never finds anything by position, since the common tenant grows all run.

**A port lands at the loosest level its reads allow,** and moving a test down a level later is ordinary work, the
expected way the tier gets faster. The [cross-check](#cross-check) catches a test placed too loose.

## Test data

Fixtures build every test's data through the surfaces above. `blizzard-mock-data` writes live into a running hub's
SQLite, which suits agents and people testing by hand; it never seeds a test in this tier. Store access is allowed only
in a `fresh-hub` test whose subject is the store, against files before the daemon starts or after it stops.

## The rulebook

blizzard-context changes in a companion that lands with this epic. It redefines `blizzard:service-test` as the
Playwright package plus the pytest cases that stay, in its command, its tier roster, and its `mise` task. It says that
new daemon tests are written in Playwright. It scopes the statement-guard rule to every daemon a test spawns rather than
to rows that run pytest, and narrows the rule that test data comes from `blizzard-mock-data` so it no longer binds this
tier. The registry check learns to attribute a Playwright spec as firmly as it attributes a pytest file, the frontend
toolchain and comment rules extend to the new folder, and the hub's pre-push sweep looks there too. A companion rule
requires every port to move its citations in the same change.

The companion also writes down the tier's own architecture, because a fleet worker arrives knowing how Playwright tests
are usually written, and most of that habit is wrong here. Its documentation and the examples an agent has absorbed
reach for the built-in `request` fixture, write full URLs, wait with `waitForTimeout`, and set up state in a
`beforeAll`; every one of those breaks a shared hub. The rules state the tier's invariants, each with the reason it
exists:

- **Everything comes from a fixture.** A test names the fixtures it needs, its tenant, project, clients, people, and
  event stream, never builds one itself, and never uses Playwright's built-in `request`.
- **Every call goes through the harness's clients.** No raw `fetch` against a daemon, no hand-built URL, and a route a
  test needs that a client lacks is added to the client, not worked around.
- **Isolation is declared and earned.** A test declares its level, the loosest its reads allow, and keeps the read rules
  at `common` and `fresh-project`: unique names, assertions narrowed to its own ids, no counts or absences across the
  tenant.
- **Waits end when the system speaks.** A test waits on an event or a state it can read, never on the clock; a negative
  assertion waits on a barrier act only it would emit.
- **Public surfaces only.** No store access outside a `fresh-hub` test whose subject is the store, and no seeding
  through `blizzard-mock-data`.
- **Mock state stays the test's own.** A lever touches only the test's own repository, issue, or runner.
- **Configuration is a profile.** A hub-wide setting rides a TOML profile or a fresh hub, never a test that changes it
  in place.

The rules are kept beside an exemplar spec, a ported test chosen as the reference to pattern new ones from, in
blizzard-context's exemplars. What the harness can enforce, it enforces as well: a lint rule bans the built-in `request`
fixture and `waitForTimeout` in specs, the clients fail a test on any server error, and the cross-check notices a test
that breaks the rest. The rules grow as the ports teach them: a practice a later epic settles that a capable agent would
not arrive at on its own joins the companion in the change that settles it.

## Cross-check

A test that shares can pass for a reason that is not its own: a neighbour's activity nudges a sweep along, or a
neighbour's item happens to satisfy a list it reads, and the assertion it was waiting on comes true by accident. Such a
test is in the wrong bucket, and it would fail the day it ran alone. The harness can raise every test one isolation
level by a flag, and a scheduled weekly workflow and the release workflow run the suite both ways, at high parallelism
so every test meets a crowd of neighbours; the two runs must pass the same tests. It never runs on a pull request, where
it would double the tier for a fault that surfaces slowly and belongs to the tests rather than to the change.

## Measurements owed

`initiative:multi-tenancy` left two numbers to this epic, both taken on the CI runner's shape, four vCPUs, because a
ratio found on a twenty-core laptop would not hold there.

- **Hubs per worker.** How many workers a shared hub serves before its one core becomes the wait, with and without the
  statement guards' cost, which falls on that core, over a whole run, so the sweeps of the tenants it accumulates are
  counted.
- **Store contention.** How many concurrent tenants a SQLite hub carries before tests queue behind each other's writes.
  The shared hubs run on SQLite unless the slowest store waits at the chosen ratio come within reach of the five-second
  busy timeout; then they move to Postgres, which also amends the harness rule that tests run on SQLite and adds a
  database service to the CI jobs.

## Proof

`test_analytics_service.py`, `test_analytics_events_service.py`, and `test_analytics_operational_service.py` are the
cleanest hub-side tests and move first, into `tests/service/hub/`, their Python originals deleted in the same change.
They exercise the harness end to end: a profile, a tenant, the forge, an ingest, and fleet traffic under a runner token.
Their aggregate assertions and runner traffic likely put them at `fresh-tenant`, so they prove the expensive level
first; each is placed by the rules above rather than assumed, and the first test the harness finds at `common` or
`fresh-project` proves the cheaper levels.

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

This epic builds the last two of these. The first is built here only if the measurement finds one shared hub's core
saturated by tenant creation. The spike's third finding, a misspelled key booting silently, is answered by the harness's
own key check, not by the hub.

- **Cheaper tenant creation**, such as minting packaged graphs lazily, which an operator creating tenants would feel
  too.
- **An environment variable for the CLI's tenant**, beside `BZ_HUB_URL`.
- **Unhandled request errors in the hub's own log**, tagged with the request's tenant, where today uvicorn writes them
  as plain text that names no tenant, so a guard's refusal inside a request can be read out of a shared hub's log.

Settings that stay hub-wide — route-token and produces enforcement, authentication, sweep intervals, forge write closure
— ride profiles or fresh hubs; none of them becomes per-tenant for the sake of the tests.

## Landing order

The epic lands as six children. The first three are independent of one another and of everything after them.

1. **Repositories on demand** in the mock forge, in `blizzard-mock`.
2. **The CLI's tenant variable**, beside `BZ_HUB_URL`.
3. **Unhandled request errors in the hub's own log**, as JSON tagged with the request's tenant.
4. **The daemons' environment contract**, lifted into the one place both suites read, with the pytest suite reading it
   first so the move changes nothing it runs.
5. **The Playwright package and the shared-hub harness**: profiles, the common tenant, the isolation option and the
   fixtures that resolve at all four levels, the clients and their ledger, and the CI wiring: the `playwright` task, its
   two shards beside the pytest job under the existing required check, the daemons the pinned toolchain, failure
   artifacts, and `test_winter_free_ci.py` taught to see it. The rulebook companion lands with it. It waits on 1, 3,
   and 4.
6. **The three analytics ports and the measurements**, the Python originals and their citations deleted in the same
   change, and the measured ratio and store decisions recorded here, cheaper tenant creation among them if the ratio
   asks for it. The cross-check's flag and its weekly and release runs land with it. It waits on 5.

## Done

The three analytics files run in the Playwright package under `blizzard:service-test`'s required check, in two shards in
both workflows, and their Python originals and blizzard-context citations are gone. Every daemon the project starts runs
under the shared environment contract. The mock forge creates repositories on demand, and the cross-check runs weekly
and in the release workflow. The two measurements are recorded with the decisions they drove, the tier's CI time before
and after is recorded beside them, and the rulebook companion has landed.
