# A host from files

## What the contract covers

`contracts/config/host.json` documents the host file's keys and seven paths: `.env`, `blizzard-host.toml`, `data`,
`data/host.db`, `data/host.lock`, `host.sock`, `runners`. It says nothing of what lives inside `runners/` or `.env`, and
nothing of the harness files `blizzard-runner init` writes, so a test that lays out a host today does it through
`tests/host_files.py`'s `init_runner_environment`, `host_config_of`, and `write_host_file`, `HubTokenFile`, and
`write_runner_id`.

The contract's `paths` grow to name every file a host directory holds, each with what writes it and its mode where it
matters:

| Path                            | What it is                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------- |
| `.env`                          | The host's tokens, mode `0600`, written atomically; its variables are below                  |
| `runners/<name>/`               | One directory per runner, created by `add`, reserved for that runner's state and empty today |
| `worker-settings.json`          | The Claude Code worker settings `init` writes, carrying the heartbeat and session-end hooks  |
| `plugins/blizzard-heartbeat.ts` | The OpenCode heartbeat plugin `init` writes                                                  |
| `opencode-worker-config.json`   | The OpenCode worker configuration `init` writes                                              |

The contract gains an `env` section naming the variables `.env` holds: `BZ_HOST_TOKEN`, the host's token, and
`BZ_HUB_TOKEN_<NAME>`, each runner's, where `<NAME>` is the runner's name upper-cased with every non-alphanumeric
character replaced by `_`, the rule `src/blizzard/foundation/runner_token_names.py` implements.
`blizzard-export-contracts` writes both from the code that owns them, and `tests/test_config_contract.py` holds them to
it, as it holds the keys.

## How the harness builds a host

The `runnerHost` fixture is a host built for the test alone, from the runner profile beside it, so a test names
`({ runnerHost })` and never lays one out. It is built in four steps:

1. It mints a fixture world with `blizzard-mock-fixture mint` into the test's directory, which gives it origins and a
   workspace, and writes `.blizzard-mock-harness-fence` there.
2. It runs `blizzard-runner init <dir>` with `BZ_HUB_URL` naming the test's hub: the test's own mock hub in
   `service/runner/`, the shared hub in `service/fleet/` in `tenant`. `init` writes the host file, the harness files,
   and `.env`, and registers the host and its first runner, as an operator's install does.
3. It merges the test's runner profile over the host file `init` wrote, as a hub profile merges
   ([the foundation](../../foundation/spec/hubs.md#profiles)), every key checked against `contracts/config/host.json`'s
   keys first. A profile here may expand `${workspace}`, `${origins}`, and `${mockBin}`, which name the minted world's
   paths and the mock fleet's binaries, so it can point the harnesses at `mock-claude-code`, `mock-opencode`, and
   `mock-codex` and the workspace at the minted one. Additional runners the profile names join with
   `blizzard-runner add`.
4. It starts `blizzard-runner host --dir <dir> --loop stepped`, unless the test asks for a timed host, with the
   foundation's daemon environment, `BLIZZARD_MOCK_HARNESS_FENCE=1`, and that variable in `worker.env_passthrough`, so a
   worker that escapes the fixture world exits rather than touching the developer's own.

A test that needs a layout no verb produces, such as a halted runner, a foreign runner in the store, or a token file
missing, writes the documented files directly after step 3, by the contract, never through a class of the runner's. A
runner profile lives beside the test as `runner.toml`, or inline through `test.use({ runnerConfig })`, since the host is
always the test's own.

## The fleet folder

In `tests/service/fleet/`, the hub is real and shared, so `init` joins the shared hub in `tenant`, which needs a runner
token and the host plane, and the test is at `fresh-tenant` at least: its runner claims tenant-wide work. A test whose
subject is hub-wide takes `fresh-hub`, as the plan names: `runner_init`'s reset of the hub's data, and
`idp_federation`'s provider changes and key rotation.

`runnerHost` holds `{ dir, url, socket, runners, step(runner?), stepHost(), stop(), start() }`: `step` and `stepHost`
call [the step routes](./stepping.md#the-step-routes) over `host.sock` and resolve with their answer, and `stop` and
`start` give the test its host's lifecycle. A test needing a second host, such as a lock held by another, calls
`runnerHosts.create(options)`, a factory fixture that builds one the same way.
