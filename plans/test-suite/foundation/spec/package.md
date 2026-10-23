# The Playwright package

## Files

The package is rooted at `tests/`, beside the Python suites it will replace, and is one npm package with one Playwright
config:

| Path                             | What it holds                                                                                                                                                |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `tests/package.json`             | `@playwright/test`, `typescript`, `eslint` with `typescript-eslint`, `@hey-api/openapi-ts`, and a TOML library; scripts `typecheck`, `lint`, `generate:wire` |
| `tests/package-lock.json`        | Committed; every install is `npm ci`                                                                                                                         |
| `tests/playwright.config.ts`     | The projects, derived from the folders below, and the run-wide settings                                                                                      |
| `tests/tsconfig.json`            | `strict`, `noEmit`, covering `harness/` and every `*.spec.ts`                                                                                                |
| `tests/eslint.config.mjs`        | The package's lint, including the bans [rulebook.md](./rulebook.md#what-the-harness-enforces) names                                                          |
| `tests/openapi-ts.config.ts`     | The wire-type generation ([below](#wire-types))                                                                                                              |
| `tests/daemon-env.json`          | The daemons' environment contract, read by both suites ([daemons.md](./daemons.md#the-environment-contract))                                                 |
| `tests/harness/`                 | Everything that is not a test: fixtures, daemon control, profiles, the clients, the event-stream reader                                                      |
| `tests/service/hub/**/*.spec.ts` | The hub's service tests, and its profiles as subfolders ([hubs.md](./hubs.md#profiles))                                                                      |
| `tests/.gitignore`               | `node_modules/`, `test-results/`, `playwright-report/`, `.run/`                                                                                              |

`tests/harness/fixtures.ts` exports the package's `test` and `expect`, extended with every fixture this spec names, and
a spec imports from it and from nothing else in the harness. The rest of `harness/` is internal to it:

```text
tests/
├── package.json · package-lock.json · tsconfig.json · eslint.config.mjs
├── playwright.config.ts          projects derived from service/hub/ and its profile subfolders
├── openapi-ts.config.ts
├── daemon-env.json               read by harness/env.ts and by tests/suite_plugin.py
├── harness/
│   ├── fixtures.ts               ◀ the only module a spec imports: test, expect
│   ├── tenancy.ts                level, tenant, project, and the hub a test reaches
│   ├── factories.ts              issue, repository, mockRunner
│   ├── client/
│   │   ├── clients.ts            api, fleet, forge
│   │   ├── resources.ts          what the ledger records and narrows
│   │   ├── ledger.ts             what the test created
│   │   ├── mock.ts               hand-typed mock-fleet shapes
│   │   └── wire/hub/             generated from openapi/hub.openapi.json
│   ├── events.ts                 events
│   ├── logs.ts                   hubLog, and scanning for refusals
│   ├── daemons/                  hub.ts · forge.ts · idp.ts · mock-runner.ts
│   ├── env.ts
│   ├── profiles.ts
│   ├── setup/                    setup-<project> and teardown-<project> specs
│   ├── checks/                   the harness's own proof, project `harness`
│   └── scripts/                  cross-check.ts
├── service/
│   └── hub/
│       ├── profile.toml          the default profile → project service-hub
│       ├── *.spec.ts
│       └── <profile>/
│           ├── profile.toml      → project service-hub-<profile>
│           └── *.spec.ts
└── .run/<run-id>/                daemons' directories and logs, hubs/<project>.json
```

| Module                 | Owns                                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `harness/daemons/`     | Starting and stopping each daemon: `hub.ts`, `forge.ts`, `idp.ts`, `mock-runner.ts`                                                   |
| `harness/env.ts`       | Reading `daemon-env.json` into a child environment                                                                                    |
| `harness/profiles.ts`  | Loading a profile and checking its keys                                                                                               |
| `harness/setup/`       | The setup and teardown projects' specs                                                                                                |
| `harness/tenancy.ts`   | The isolation option and the `level`, `tenant`, and `project` fixtures, and which hub a test reaches                                  |
| `harness/factories.ts` | The `issue`, `repository`, and `mockRunner` factory fixtures                                                                          |
| `harness/client/`      | The clients, the ledger and the `resources.ts` map it reads, and `wire/`, the generated types                                         |
| `harness/events.ts`    | The event-stream reader                                                                                                               |
| `harness/logs.ts`      | Reading a daemon's log, by tenant, and scanning it for refusals                                                                       |
| `harness/checks/`      | The harness's own proof, a spec that exercises every level against real hubs ([carry-over.md](./carry-over.md#the-order-of-landings)) |
| `harness/scripts/`     | Scripts the workflows run over a run's reports, `cross-check.ts` among them                                                           |

The `service/runner/`, `service/fleet/`, and `ui/` folders do not exist until the epics that fill them create them, and
nothing in the config refers to them in advance.

## The fixtures

A spec names the fixtures it needs ([isolation.md](./isolation.md#the-fixtures) owns what each is) and builds none of
them. Each fixture below holds the fixtures it uses, the way a folder holds its files. They are drawn in the order they
build on one another, and a fixture already drawn appears again as `↑`:

```text
level                        test    { declared, served }
└── isolation                option  set with test.use

testHub                      test    internal: the hub this test reaches, and the worker's request context for it
├── level ↑
└── hub                      worker  this worker's shared hub, by parallelIndex; asked only below fresh-hub
    └── setup-<project>      —       the project dependency that wrote hubs/<project>.json

tenant                       test    the tenant at the served level
├── level ↑
├── testHub ↑
└── adminClient              worker  internal: creates tenants and makes other hub-level calls
    └── hub ↑

api                          test    the hub client, as the tenant's admin
├── tenant ↑
├── level ↑
├── testHub ↑
└── ledger                   test    internal: every id the test creates

project                      test    the project at the served level
├── tenant ↑
└── api ↑                            creates it at fresh-project

fleet(name)                  test    a fleet-plane client under a runner token
├── api ↑                            mints the token
└── ledger ↑

forge                        test    the hub's mock forge
├── testHub ↑
├── level ↑
└── ledger ↑

events                       test    the event stream, narrowed to what the test made
├── tenant ↑
├── testHub ↑
└── ledger ↑

hubLog                       test    the hub's log, filtered to the tenant
├── tenant ↑
├── level ↑
└── testHub ↑

issue(title)                 test    factory: files an issue and ingests it
├── forge ↑
├── api ↑
└── project ↑

repository(name?)            test    factory: a repository and its config/apply, fresh-tenant and up
├── forge ↑
├── api ↑
└── project ↑

mockRunner(name)             test    factory: a mock runner under a token in the tenant, fresh-tenant and up
├── api ↑
└── testHub ↑
```

At `fresh-hub`, `testHub` starts a hub for the test alone through `harness/daemons/` and never asks for `hub`, so the
test touches no shared hub, and `tenant` is that hub's seeded one, needing no `adminClient`.

A test names only the top of what it needs: `({ api, project })` gets the tenant, the hub, and the ledger beneath them
without naming any. The later epics add fixtures the same way: `user`, `people`, `cli`, `freshHub`, and `otlpSink` from
the hub tier, and `runnerHost` from the runner tier.

## Projects

A Playwright project is a test folder crossed with a hub profile. `playwright.config.ts` walks `tests/service/hub/` once
at load: the folder's root is the default profile, project `service-hub`, and each subfolder holding a `profile.toml` is
a profile of its own, project `service-hub-<subfolder>`, whose `testDir` is that subfolder. The default project's
`testIgnore` excludes every profile subfolder, so a spec runs under exactly one project, the one where it lives. A
subfolder without a `profile.toml` is an ordinary grouping of tests and belongs to the profile above it.

One project stands apart from the folders: `harness`, whose `testDir` is `harness/checks/`, runs on the default
profile's setup and is selected by name, never by `service-*`.

Each profile project has a setup project, `setup-<project>`, as its dependency, and that setup project names
`teardown-<project>` as its teardown ([hubs.md](./hubs.md#setup-and-teardown)). A run that selects no test in a profile
runs neither, so `--project=service-hub-auth` starts no default-profile hub. A `fresh-hub` test is selected with its
folder's project, so a run of that one test still runs the project's setup and starts its shared hubs, about ten seconds
a developer pays once.

Run-wide settings: `fullyParallel: true`; `forbidOnly` and `retries: 0` under `CI`; no `use.baseURL`, because a test
reaches its hub only through fixtures; reporters `list` and `html`, plus `json` to `test-results/report.json` under
`CI`; `outputDir` `test-results/`. The service projects install no browser and set no `browserName`. The worker count is
the config's `workers`: two under `CI` to start, moved only by [the measurement](./carry-over.md#the-measurements), and
overridable on the command line.

## Toolchain

The daemons run as the installed binaries an operator runs: `blizzard-hub` from `.venv/bin/` after `uv sync`, and the
mock fleet's `blizzard-mock-forge`, `blizzard-mock-idp`, and `blizzard-mock-runner` from the sibling
`../blizzard-mock/.venv/bin/`, the directory `tests/fleet_launchers.py`'s `mock_bin_dir` already resolves. Either
directory can be overridden, by `BLIZZARD_BIN_DIR` and `BLIZZARD_MOCK_BIN_DIR`. A missing binary fails the setup project
with the message the pytest suite skips with, naming `winter provision <env>`. The tier never skips, since a skipped
service tier reads as a green one.

Node is the version `mise.toml` pins, made exact by this epic (`node = "22.<minor>.<patch>"` in place of `"22"`, at
22.13 or later, the first 22 release to load `node:sqlite` without a flag, which the runner tier's store tests use), and
`tests/package.json` declares the same in `engines`. Playwright's own version is pinned exactly, never by range, so the
lockfile and the reports agree on it.

## Wire types

`tests/openapi-ts.config.ts` generates TypeScript types from `openapi/hub.openapi.json` into
`tests/harness/client/wire/hub/`, with the `@hey-api/typescript` plugin only: types, not a client, since the calls go
through the clients. The output is committed, as `web/`'s generated client is. `npm run generate:wire` regenerates it,
and the static job fails when regenerating changes anything ([ci.md](./ci.md#the-static-job)), so a renamed field in the
hub's OpenAPI document breaks `tsc` in the package before it breaks a run. The runner tier adds `runner.openapi.json` to
the same config when it needs it.

The mock fleet publishes no OpenAPI document. The harness types the few lever and control shapes it uses by hand, in
`harness/client/mock.ts`, and only those.

## Beside the Python suites

`tests/` stays a Python package until the last Python suite has left it, and every Python tool that walks it learns to
skip what the npm package adds:

- **pytest** already skips `node_modules` through its default `norecursedirs`, and `pyproject.toml` writes the list out
  explicitly with `node_modules` in it, so a later edit to the setting cannot drop it silently.
- **pyright**'s `exclude` gains `tests/node_modules`.
- **The repo-read guard** behind `tests/repo_files.py`'s `audited_roots` exempts `tests/node_modules` and `tests/.run`,
  as it exempts `.venv`.
- **dprint** excludes `tests/node_modules`, `tests/harness/client/wire`, and `tests/package-lock.json`.

The pytest half of the tier keeps its files in `tests/service/` beside the `hub/` folder, and its runs are unchanged.
