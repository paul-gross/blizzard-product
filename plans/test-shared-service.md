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

- **Hub-wide startup config** (auth mode, route-token mode, runner-auth mode, produces mode), unless multi-tenancy makes
  it per-tenant. Handle this with a pool of shared hubs, one per config profile.
- **Anything that stops, kills, restarts, or migrates the hub,** including the crash sweep and restart/resume tests.
- **Tenant administration,** and whatever multi-tenancy leaves global.

## Work areas

### 1. Shared hub harness

- **Hub lifetime:** one long-lived hub per xdist worker or per CI job, for each config profile.
- **Tenant per test:** create it via the admin API, together with its runner credentials, graphs, projects, and work
  sources. The test acts only with that tenant's identity.
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
