# A host that steps on request

## The configuration

`blizzard-host.toml` gains a top-level key, `loop`, one of `"timed"`, the default and today's behaviour, or `"stepped"`,
declared in `blizzard.runner.config_keys` so it reaches `contracts/config/host.json`.
`blizzard-runner host --loop stepped` overrides the file for one run, for an operator who wants to freeze a misbehaving
runner without editing it.

On a stepped host, nothing that runs on a timer today starts its timer: no runner's `PeriodicDriver`, no `HostTick`'s
`PeriodicPassDriver`, and no trace sweep. Everything else is as on a timed host: the local API is served, startup runs
each runner's `resume.on_startup()` and reports the host once through `host.host_tick.report()`, so the host is
registered before the first step, and workers a step spawns keep running between steps, calling back into the API as
they do between timed passes. Shutdown is unchanged.

## The step routes

Two routes run a pass. Each holds its request open for the whole pass and answers only once the pass has completed, so a
caller that gets its answer knows every phase has run and every effect the pass had is readable:

| Route                                 | Runs                                                                                                                                                                                                                                                                         |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /runners/{runner_id}/api/ticks` | One `tick` for that runner, every phase a timed pass runs: liveness, identity, spend ceiling, then REAP → RESUME → PULL → complete held bindings → FILL → ADVANCE → transcript drain, then retention and context sampling; and, when tracing is on, one trace sweep after it |
| `POST /api/host/ticks`                | One `HostTick` pass: registration, credential renewal, usage sampling, lane drain                                                                                                                                                                                            |

Each answers `200` with `{ "started_at", "finished_at", "outcome": "completed" | "failed", "error"? }`. A pass that
raises answers `200` with `outcome: "failed"` and the error's message, since a timed pass's failure is logged and
swallowed and a stepped one reports the same fact to its caller rather than becoming a server error. Both are on the
human lane, as `PATCH /runners/{runner_id}/api/runner` is: free over `host.sock`, gated over TCP when the hub offers an
IdP.

Both routes serve on a timed host too. A runner's pass, timed or requested, holds a per-runner step lock for its length,
and a second request arriving while it is held answers `409` with `{ "code": "tick_in_progress" }` at once rather than
queueing behind it, so a caller never waits on a pass it did not ask for, and learns at once that it raced. A test owns
its host and steps it in sequence, so it meets the refusal only by a defect of its own. `HostTick` has its own lock and
the same refusal. The lock is in memory, in the host process, and ends with it.

A requested tick runs on the request's thread through the runner's existing `LoopWiring`, so it uses the same graph, hub
client, and broker as the timed driver would; `tick_once`'s per-call construction stays for the component tier, which
calls it in-process.

## `blizzard-runner tick` against a running host

`tick_cmd` today refuses when `data/host.lock` is held. It probes first instead: when the lock is held and `host.sock`
answers `GET /api/health`, it calls `POST /runners/{runner_id}/api/ticks` over the socket for the runner `--runner`
names, or the sole runner, prints `tick complete` or the failure, and exits `0`, or `1` on `failed`, `tick_in_progress`,
or a refusal. When the lock is held and the socket does not answer, it refuses as today. When the lock is free, it runs
the pass itself, as today.

`blizzard-runner tick --host` calls `POST /api/host/ticks` the same way, and is refused when no host is running.

## Proof

`openapi/runner.openapi.json` carries both routes, and `blizzard-context`'s architecture (`bzh:steppable-loop`, in
`architecture/crash-correctness.md`) names them as the loop's outside handle
([carry-over.md](./carry-over.md#citations-and-the-rulebook)). The component tier covers the step lock's refusal and a
failed pass's answer in-process; the service tier covers a stepped host end to end through every port that steps.
