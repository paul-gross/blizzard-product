# The hubs

## Profiles

A profile is the hub configuration its folder's tests run against, written as `profile.toml` in that folder:
`tests/service/hub/profile.toml` for the default profile, `tests/service/hub/<name>/profile.toml` for each other. It
holds only the keys that differ from what `blizzard-hub init` writes, in the same TOML shape as `blizzard-hub.toml`, and
the default profile may be empty. A hub starts by running `init`, then merging its profile over the file `init` wrote,
table by table and key by key, then hosting. `init` owns everything a profile does not name, the store's location and
the secret key among them, so a profile never restates them.

The hub reads its configuration leniently by key, and a key it does not declare loads without complaint, so a misspelled
setting in a profile would leave every test on it believing in a configuration the hub never ran. `harness/profiles.ts`
checks every profile before any hub starts. It flattens the profile into dotted paths, `[auth]` and `superuser` becoming
`auth.superuser`, and each path must be a `keys[].path` in `contracts/config/hub.json` that is not `deprecated`. A
value's TOML type must match the key's declared `type`. The first setup project of a run checks every profile in the
package, not only those its run selected, so a broken profile fails the run that introduced it whatever that run
selected. The check is the harness's own, permanent, and asks nothing of the hub, which still refuses a bad value at
boot.

A profile names no forge and no work source. Those are a tenant's configuration, furnished by the `repository` fixture
through `POST /api/config/apply` ([isolation.md](./isolation.md#what-the-test-makes)), so one profile serves tests on
any forge.

Settings that stay hub-wide ride a profile or a fresh hub: authentication and its providers, route-token and produces
enforcement, sweep and annotation intervals, forge write closure. The foundation's three ports run on the default
profile, and the hub tier adds the authentication profile.

## Shared hubs

A profile's setup project starts its shared hubs, all at once, and each shared hub is a hub, a mock forge, and a stub
IdP of its own, so that one hub's tests never race another's on mock state. Neither mock runs migrations, so the pair is
cheap beside the hub.

How many hubs a profile gets is `hubsFor(profile)`, computed in `playwright.config.ts` from the run's worker count:

- **The default profile** gets `ceil(workers / WORKERS_PER_HUB)` hubs. `WORKERS_PER_HUB` is a constant in the config,
  set to `8`, so a run of up to eight workers shares one hub. [The measurement](./carry-over.md#the-measurements) checks
  that start rather than choosing it.
- **Any other profile** gets one hub, and its project sets `workers: 1`, so one worker runs its tests in sequence
  against it. A hub-wide setting only a few tests need costs one hub start, and only in a run that selects one of them.

A worker reaches its hub by its `parallelIndex`, the stable slot Playwright gives each concurrent worker, never by
`workerIndex`, which grows each time a worker restarts after a failure: hub `floor(parallelIndex / WORKERS_PER_HUB)` of
the default profile. A worker that restarts lands back on the hub its slot always used. `parallelIndex` counts across
the whole run, not per project, so the one worker of any other profile's project may hold any slot, and reaches its
profile's one hub, hub `0`, whatever its slot.

The `hub` fixture is worker-scoped and hands a test its hub's coordinates: the hub's URL, its forge's and IdP's, the
common tenant's id, and its log's path. It reads them from `.run/<run-id>/hubs/<project>.json`, which the setup project
writes, so no worker starts a shared daemon of its own.

Each hub has one request context per worker, created with the hub's URL and disposed when the worker ends. A browser's
cookie jar ignores the port, so two hubs on one host would otherwise hand each other's sessions back. A sign-in's
session lives in a context of its own, created for that person, as the hub tier builds.

## Setup and teardown

`setup-<project>` is a project of its own, whose one spec in `harness/setup/` does in order:

1. On the run's first setup project only: the check of every profile.
2. Starts the profile's hubs, each with its forge and IdP, concurrently.
3. On each hub, creates the common tenant ([below](#the-common-tenant)).
4. Writes `.run/<run-id>/hubs/<project>.json`.

The first-only steps are made safe for concurrent setup projects by an exclusive create of `.run/<run-id>/first.lock`:
the one that creates it runs them and writes `first.done`, and the others wait on that file.

`teardown-<project>` stops every daemon its setup started and reads their logs for
[refusals](./daemons.md#refusals-attributed). Playwright runs it after the last test in the profile's project, and still
runs it after a failure.

## The common tenant

Every shared hub holds two open tenants once its setup project has run: the tenant its first migration seeds, which a
single-tenant hub answers header-less requests with, and the common tenant, created by `POST /api/admin/tenants` with
the name `common`. With two open, the hub answers a request that names no tenant with `409 tenant_required`, so a client
that forgot the header fails on the first call rather than landing somewhere.

The common tenant is furnished once, by the setup project, with what a test needs to make work in it and nothing more: a
repository `blizzard/common` on the hub's forge, created as [surfaces.md](./surfaces.md#repositories-on-demand)
describes, and one `POST /api/config/apply` declaring that forge as the tenant's work source, the repository, and its
link to the `default` project. Chunks come into being only by ingest from a work source, so without these a test at
`common` could make nothing at all. It holds no secret, no runner, and no second repository, and nothing furnished there
is ever changed again: a test that needs a repository of its own, a lever on one, or a runner is at `fresh-tenant`
([isolation.md](./isolation.md#what-the-test-makes)).

A test at `common` or `fresh-project` makes its work by filing an issue of its own in `blizzard/common` and ingesting
it, the `issue` fixture doing both, and owns the issue and the chunk that come of it. Issue numbers count per
repository, so every test's issue is its own, and nothing it does reaches another test's.

The shared hubs run with authentication `none`. The harness acts as the implicit operator `resolve_tenant` gives an
unauthenticated request, an admin and a hub admin, and names its tenant in `X-Blizzard-Tenant`. Only the harness's own
fixtures make the hub-level calls, tenant creation among them, through an internal client no spec can reach.

## Fresh hubs

A test at `fresh-hub` gets a hub started for it alone, with a forge and IdP of its own, under the profile of the folder
it lives in, and stopped when the test ends. It is never written to `hubs/<project>.json` and nothing else reaches it.
No common tenant is created on it; its `tenant` is the hub's seeded one, and since that is the only open tenant, the hub
answers header-less requests too, which a test of that behaviour needs.

A fresh hub costs about ten seconds, the whole of `init` and the migrations. The harness never shortens that by starting
from a pre-migrated store ([the initiative's](../../initiative.md#decisions) black-box rule). A fresh hub's log is the
test's whole, so it can assert on any line.
