# What the epic asks of blizzard-mock and the product

## Repositories on demand

Today the mock forge serves the repositories it finds under `--repos-dir`, which the fixture mint writes before the
forge starts: `toy-api.git` and `toy-web.git`, bare and seeded with one commit on `main`. Every service test addresses
`blizzard/toy-api`, which works only because no two tests share a forge. On a shared hub every test works in a
repository of its own, and the forge has no way to make one.

`blizzard-mock` adds one test-control route to the forge, beside its levers:

```text
POST /_repos
{ "owner": "blizzard", "name": "<name>", "seed": "toy-api" }

201 { "full_name": "blizzard/<name>", "origin": "<repos-dir>/blizzard/<name>.git", "default_branch": "main" }
409 { "message": "repository exists" }
422 when owner or name is not a valid GitHub name, or seed names no known seed
```

The forge creates the bare repository at `<repos-dir>/<owner>/<name>.git`, the first path `GitBackend._candidates` looks
at, with `SubprocessGit.init_bare` and `--initial-branch=main`, and seeds it with `SubprocessGit.seed_repo` from the
named entry of `fixture_workspace/seed.py`'s `TOY_REPOS`. `seed` defaults to `toy-api`, the repository every ported test
addresses today, so a port keeps the files its original saw. The repository is served from the next request, since
`GitBackend` looks a repository up on disk at every cache miss and caches only a hit. Its issues, pulls, and labels
start empty, as any repository's do, the forge's in-memory state for it being created on first use. Creating is
idempotent only by refusal: a second create of the same name answers `409`, and the harness's names never repeat.

The route lives in a `forge/api/repos.py` router and a `ForgeService.create_repository` beside `clear_all_levers`, the
seeding reached through the forge's existing git port rather than by importing the fixture CLI. The forge's README lists
it with the levers, as test control rather than GitHub surface. `blizzard-mock`'s own tests cover the create, the
refusal, and an immediate `GET /repos/{owner}/{name}` that sees it.

Nothing else in the forge changes. Its levers are already scoped per repository, per pull request, or per branch, and
the scoping a shared hub needs is a rule of the tier's, not the forge's: a test arms levers only on its own repository,
never a lever with no `repo`, and never `POST /_levers/reset` outside a fresh hub (`bzh:service-own-mock-state`,
[rulebook.md](./rulebook.md#the-tiers-invariants)).

## The CLI's tenant variable

Tenant-scoped `blizzard hub` verbs take `--tenant` (`HubCommand.tenant_option`, `src/blizzard/hub/cli/command.py`), and
a caller who can reach several tenants and names none is refused with the list. The URL has an environment variable,
`BZ_HUB_URL`, read in `src/blizzard/hub/cli/context.py` beside its `ENV_HUB_URL` constant, but the tenant has none, so a
script or a test driving many verbs must repeat the flag on every one.

`BZ_HUB_TENANT` is read beside it: `ENV_HUB_TENANT = "BZ_HUB_TENANT"` in `context.py`, and the context's tenant is
`--tenant` when given, else the variable, else none, exactly as the URL resolves. An empty value counts as unset. The
verbs that take no tenant ignore it. The runner's own `--tenant` (`src/blizzard/runner/cli/runtime.py`) is unchanged,
since it names the tenant a runner is added to, a different act.

The CLI's reference and `docs/deployment/tenants.md` name the variable beside `BZ_HUB_URL`. `tests/suite_plugin.py`'s
`_strip_hub_coordinates_env` strips it with the other hub coordinates, and `daemon-env.json` unsets it
([daemons.md](./daemons.md#the-environment-contract)), so a developer's shell never names a test's tenant.
`tests/test_hub_cli_tenant_header.py` learns the variable's three cases: it sends the header, `--tenant` overrides it,
and an empty one sends none.

## Cheaper tenant creation

Creating a tenant mints its six packaged graphs (`src/blizzard/hub/graphs/`) in the same transaction as the tenant and
its `default` project (`TenantAdministration.create`), and on the spike that dominated the cost of a `fresh-tenant` test
and pushed the slowest calls toward the store's busy timeout. This epic builds the change below only if
[the measured ratio](./carry-over.md#the-measurements) asks for more hubs than the CI runner has cores.

If built, the packaged graphs are minted on demand rather than at creation. Tenant creation records the tenant, seeds
its sequence, creates its `default` project, and grants its admin, as now, and records that the tenant's packaged graphs
are owed. The first transaction that reads or resolves a tenant's graphs, by listing them, by creating or advancing a
chunk, or by applying configuration, mints what is owed first, idempotently, so two such transactions racing mint once.
Nothing observable through the hub's surfaces changes: every read of a new tenant's graphs returns the six packaged
graphs, at the versions creation would have minted. An operator creating many tenants feels it as creation that returns
sooner.
