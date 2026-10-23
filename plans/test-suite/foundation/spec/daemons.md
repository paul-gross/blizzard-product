# The daemons

## The environment contract

Every daemon the pytest suite spawns today inherits an environment assembled in `tests/suite_plugin.py`.
`pytest_configure` sets the tenant and runner statement guards (`BLIZZARD_TENANT_GUARD`, `BLIZZARD_RUNNER_GUARD`),
strict configuration reads (`BLIZZARD_STRICT_CONFIG`), and commit and tag signing off through `GIT_CONFIG_COUNT` and its
key and value pairs, and it pops `LEGACY_FORGE_VARIABLES`. Autouse fixtures strip the operator's tracing variables, the
worker-identity family, and the hub coordinates, and `run_hub` drops every `OTEL_` variable again on its own. The CLI's
session file is isolated only in-process, by patching `platformdirs`; a CLI run as a subprocess reads the developer's
own unless the test sets `XDG_CONFIG_HOME` by hand, as `test_host_init_readd_service.py` does.

`tests/daemon-env.json` gathers all of it into one declaration both suites read, shown here with `unset` cut short:

```json
{
  "set": {
    "BLIZZARD_TENANT_GUARD": "1",
    "BLIZZARD_RUNNER_GUARD": "1",
    "BLIZZARD_STRICT_CONFIG": "1",
    "BZ_LOG_FORMAT": "json",
    "GIT_CONFIG_COUNT": "2",
    "GIT_CONFIG_KEY_0": "commit.gpgsign",
    "GIT_CONFIG_VALUE_0": "false",
    "GIT_CONFIG_KEY_1": "tag.gpgsign",
    "GIT_CONFIG_VALUE_1": "false"
  },
  "unset": ["BZ_FORGE_URL", "BZ_FORGE_OWNER", "BZ_FORGE_BASE_BRANCH", "BZ_FORGE_TOKEN", "BLIZZARD_TRACEPARENT", "BZ_HUB_URL"],
  "unsetPrefixes": ["OTEL_", "BZ_HUB_TOKEN"],
  "isolatedConfigHome": true,
  "refusals": ["UnscopedTenantStatement", "UnscopedRunnerStatement"]
}
```

`unset` carries the legacy forge variables, `BLIZZARD_TRACEPARENT`, the hub coordinates (`BZ_HUB_URL`, and
`BZ_HUB_TENANT` once [it exists](./surfaces.md#the-clis-tenant-variable)), and the worker-identity family
`_strip_worker_identity_env` removes today, each by name. `isolatedConfigHome` asks each reader to point
`XDG_CONFIG_HOME` at an empty directory of the run's own, so no CLI a test spawns finds a session it did not create.
`refusals` names the exceptions the two guards raise (`blizzard.foundation.store.key_scoping`), which the harness
[attributes and fails on](#refusals-attributed).

The pytest suite reads the file first, in the change that introduces it. `pytest_configure` applies `set` and `unset`
from it in place of its literals, and the autouse fixtures take their lists from it, so the pytest suite runs exactly
what it ran before. A unit test, `tests/test_daemon_env_contract.py`, holds the file to the code it restates: the guard
variables are `GUARD_ENVIRONMENT_VARIABLE` and `RUNNER_GUARD_ENVIRONMENT_VARIABLE`, strict reads are
`STRICT_CONFIG_ENVIRONMENT_VARIABLE`, `unset` holds every name in `LEGACY_FORGE_VARIABLES`, and `refusals` names the two
exception classes. A guard renamed in `src/` fails that test before it silently drops out of both suites.

The harness reads it in `harness/env.ts`. A daemon's environment is the runner's own, less every `unset` name and every
variable under an `unsetPrefixes` prefix, plus `set`, plus `XDG_CONFIG_HOME` pointed at `.run/<run>/config-home/`, plus
what the daemon's start needs of its own, such as a mock runner's `BZ_HUB_URL`. Nothing a test passes can remove a `set`
variable.

On macOS `platformdirs` does not consult `XDG_CONFIG_HOME`, so the session isolation holds on Linux, which is every CI
runner. A developer on macOS running a test that drives the CLI gets the guarantee once the CLI has a session-file
override of its own, which this epic does not build, since no test it ports drives the CLI.

## Starting them

Each daemon is a child process the harness spawns with the contract's environment, its working directory under the run's
directory, and its stdout and stderr appended to one log file. A start returns once the daemon answers its health route
with `200`, or fails with the log's tail if the process exits or forty seconds pass, as `await_http` does today.

| Daemon      | Command                                                                                                                                                                    | Ready when        |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| Hub         | `blizzard-hub init <dir>`, the profile written over the TOML it produced ([hubs.md](./hubs.md#profiles)), then `blizzard-hub host --dir <dir> --host 127.0.0.1 --port <p>` | `GET /api/health` |
| Mock forge  | `blizzard-mock-forge --repos-dir <dir>/origins --host 127.0.0.1 --port <p>`, the directory starting empty                                                                  | `GET /healthz`    |
| Stub IdP    | `blizzard-mock-idp --host 127.0.0.1 --port <p>`                                                                                                                            | `GET /healthz`    |
| Mock runner | `blizzard-mock-runner --host 127.0.0.1 --port <p> --hub-url <hub> --name <name> --token <token>`, the token always minted in the test's tenant                             | `GET /api/health` |

A port is taken by binding port `0` on `127.0.0.1`, reading the port the system gave, and closing the socket before the
daemon binds it. A daemon that exits with an address-in-use error is started once more on a new port; a second failure
fails the start.

No hub start mints a fixture world. A hub's forge begins with no repositories, and every test that needs one
[creates its own](./isolation.md#what-the-test-makes). The fixture mint stays where the pytest suite and the runner side
use it.

A daemon stops with `SIGTERM`, and `SIGKILL` if it has not exited ten seconds later, as `terminate` does today.

## The run's directory

`.run/` is relative to `tests/` and shared by every run on the checkout. The run's id is minted once in
`playwright.config.ts` when `BLIZZARD_RUN_ID` is unset and written back into the environment, so the config's
re-evaluation in every worker reads the same one. A run's daemons, logs, and hub coordinates live under
`.run/<run-id>/`, and a run removes its own directory only when it passed and `CI` is unset, so a red run's logs are
still on disk to read.

## Logs

A hub's log is JSON lines under `BZ_LOG_FORMAT=json`. Every line a tenant-bound request produces carries `tenant_id`,
bound by `tenant_bound` in `src/blizzard/hub/api/auth_session.py`; lines from per-tenant background passes do not yet
carry it, and the hub tier adds the tag there. `harness/logs.ts` reads a log incrementally and offers a test its hub's
lines filtered to its own tenant. On a shared hub that is the only read it offers; a test that asserts on an untagged
line is reading a hub-wide fact and is refused below `fresh-hub` ([isolation.md](./isolation.md#reads)).

A failing test has the tail of every log it touched attached to its report, filtered to its tenant on a shared hub and
whole on a fresh one.

## Refusals, attributed

A statement guard refuses inside the daemon, and a refusal nobody reads protects nothing: on a shared hub the tenant
guard is the only thing that sees a cross-tenant statement at all. The harness therefore fails on every refusal, and
fails whoever caused it, never a test that merely ran beside it. A refusal has one of three owners.

**A request: the test that sent it.** No exception handler in `src/blizzard/hub/app.py` maps `UnscopedTenantStatement`,
so a refusal inside a request reaches FastAPI unhandled and answers `500`. The clients fail the test that sent the
request the moment they receive an unexpected `5xx` ([isolation.md](./isolation.md#the-clients-and-the-ledger)),
attaching the response and the log lines tagged with the test's tenant around it. A refusal on the request path is that
test's failure, at the line that made the request.

**A background pass in a tenant one test owns: that test.** A per-tenant pass writes its lines with `tenant_id` once the
hub tier binds it there. The harness records, in `.run/<run-id>/tenants.json`, which test each tenant it creates belongs
to. At `fresh-tenant` and `fresh-hub` the tenant is one test's, so:

- When that test ends, its `tenant` fixture's teardown reads the lines tagged with its tenant so far and fails the test
  if any names one of the contract's `refusals`.
- A pass can run after its test has ended, and a passed test cannot be failed afterwards. The teardown project reads
  every tagged line written after its test ended, and fails the run with a message naming the owning test and the line.

A fresh hub's whole log is its test's, and is read the same way when the test stops it. A runner host the test owns is
read when it stops, for `UnscopedRunnerStatement`.

**A pass in a shared tenant, or a hub-wide line: the run.** A refusal in a line tagged with the common tenant, or with
no tenant at all, belongs to no single test, and naming one would be a guess. The teardown project fails the run with
the line and its tenant, or `hub-wide`, and blames no test. Rerunning the suspects at `fresh-tenant`, as
[the cross-check](./isolation.md#the-cross-check) does weekly, narrows it to one.

The teardown project scans every log its hubs wrote, whole, so a refusal written after the last test still turns the run
red, attributed by the same three rules.
