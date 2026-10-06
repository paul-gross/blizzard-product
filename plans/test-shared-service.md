---
epic: test-shared-service
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:test-shared-service`

Each service test builds and tears down its own fixture, forge, and hub, because the hub has no unit below the
installation that a test could own. This epic runs one long-lived hub per session instead, with each test claiming its
own tenant.

A project can't provide that isolation. Graphs, the board, and much of the hub are project-independent, so tests in
different projects would still share them. The tenant is the boundary, and `epic:multi-tenancy` builds it.

## Stays on a dedicated hub

- **Hub-wide startup config** (auth mode, route-token mode, produces mode), which multi-tenancy keeps hub-wide. Handle
  this with a pool of shared hubs, one per config profile.
- **Anything that stops, kills, restarts, or migrates the hub,** including the crash sweep and restart/resume tests.
- **Tenant administration,** and whatever multi-tenancy leaves global.

## Work areas

### 1. Shared hub harness

- **Hub lifetime:** one long-lived hub per xdist worker or per CI job, for each config profile.
- **Tenant per test:** create it via the admin API. It arrives holding the packaged graphs and a `default` project; the
  test adds its runner credentials and work sources, and acts only inside that tenant, naming it on every request.
- **Teardown:** the test deletes its tenant, but correctness can't depend on it; a leftover tenant is invisible to
  others by construction.
- **Forge and fixture:** one forge process for all tests, with per-test repositories minted inside one shared fixture
  world.

### 2. Classify the service tier

Mark each test as shareable or dedicated, using the categories above. Migrate the shareable ones one file per issue.

### 3. Leak guard

- **Canary:** a test in its own tenant asserts that no event, row, queue entry, or graph from another tenant reaches it
  while other tests run.
- **Cross-check:** run the shared suite in random order at high parallelism, and periodically compare its results
  against isolated runs on dedicated hubs.

## Dependencies

- `epic:multi-tenancy`: the hub slice for hub-side tests, the runner slice for runner-side tests.
- `epic:test-architecture`: a hub that outlives its tests can only be reached through public surfaces. As a side effect,
  the suite can run against any running hub.

## Open questions

- How many tests one SQLite hub can carry at once. Tenancy keys rows in one database rather than giving each tenant a
  store of its own, so every tenant on a hub shares one writer, and the suite's parallelism is bounded by how long tests
  queue behind each other's writes. That needs measuring before the epic promises hundreds of concurrent tests, and it
  decides whether the shared hubs run on SQLite at all.
- How a test on an auth-on profile gets a person. Only a hub with auth off lets a test act in a tenant with no user or
  session. With auth on, the hub is invite-only and admitting someone runs through a provider's sign-in round trip, so
  nothing mints a user and a session for a test. That profile needs a test-only way in — a mock provider, say — or its
  tests stay on dedicated hubs.
- How fast a tenant tears down, and whether the suite relies on it. `epic:multi-tenancy` promises a teardown fast enough
  to run after every test and leaves the number to this epic, but nothing here measures it yet. The harness above says
  correctness can't depend on teardown; whether teardown time is part of each test's budget, or only hygiene that runs
  behind the suite, decides how much it matters.
