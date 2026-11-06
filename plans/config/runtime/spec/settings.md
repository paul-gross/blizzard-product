# Settings and their lifetimes

Every setting of both daemons, its lifetime — how long a loaded value is lived with before the setting is evaluated
again — and where the running daemon holds the value today. A key's path is as it appears in its daemon's contract
document (`contracts/config/hub.json`, `contracts/config/host.json`); deprecated keys are left out, since they are read
only to be refused, graduated, or honored. Code paths are under `src/blizzard/`.

## Lifetimes

Each key's registry entry declares one of the values below, and the contract documents carry it
([lifetimes.md](./lifetimes.md)). A key whose lifetime is `pass` or `session` is an **entry**: it lives in its own
daemon's store, is written only through its own daemon's API, and is seeded from the file (§Seeding from the file). A
`boot` key is not an entry: it stays in the file, read at boot as today. An `immutable` key is an entry written once,
when the thing it belongs to is created, and refused by every later write. Neither daemon's API reads or writes the
other's entries.

| `lifetime`  | Meaning                                                                                                                                                                                                                                                                                       |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pass`      | Loaded once at the start of each pass of the loop that uses it, as one snapshot of every entry that pass reads, and held until the pass ends. A setting a hub request reads is loaded when the request arrives. Nothing is re-read mid-pass; a write made during a pass reaches the next one. |
| `session`   | Handed to a worker when its session launches. A running session keeps the value it launched with; every session launched after the change gets the new one.                                                                                                                                   |
| `boot`      | Evaluated when the daemon starts, lived with until it stops: the daemon's footing. Stays in the file; never an entry, never written through the API. Changing it is a file edit and a restart.                                                                                                |
| `immutable` | Evaluated once, when the thing it belongs to is created, and never again, not even across a restart. Changing it means creating a different thing.                                                                                                                                            |

A table key takes the lifetime of its children; a key whose children differ is listed child by child. A value with no
lifetime is not a setting: §Pinned lists the constants that stay in code.

## The one new setting

| Setting        | Daemon | Where it lives                                                                                              | `lifetime` | Default | Replaces                                               |
| -------------- | ------ | ----------------------------------------------------------------------------------------------------------- | ---------- | ------- | ------------------------------------------------------ |
| `tick_seconds` | runner | a host store entry, host level; drives every runner's tick and the host tick (`runner/cli/runtime.py:1203`) | `pass`     | `30.0`  | `BZ_RUNNER_TICK_SECONDS` (`runner/cli/runtime.py:140`) |

`BZ_RUNNER_TICK_SECONDS` is deprecated: until it is removed it seeds the entry only when neither the store nor the file
holds one, and its removal is a follow-up `(BREAK)`. The tests that shorten the tick through it — the crash suite, the
runner and journey service tests, the rehearsal installer, the runner-host harness checks — move to seeding
`tick_seconds` from the file. Every other setting below already exists as a file key; this epic changes where it lives
and when it is evaluated, not whether it exists.

## What the epic builds

The machinery that holds settings, serves them, and says which ones are in force. None of these is a setting.

| Piece                 | Daemon | Owned by                                                                                                             |
| --------------------- | ------ | -------------------------------------------------------------------------------------------------------------------- |
| `lifetime` and `env`  | both   | [lifetimes.md](./lifetimes.md)                                                                                       |
| Runner settings store | runner | [runner-store.md](./runner-store.md)                                                                                 |
| Runner settings API   | runner | [runner-api.md](./runner-api.md)                                                                                     |
| Settings revisions    | runner | [runner-store.md](./runner-store.md), [passes.md](./passes.md) §Sessions — kept on the runner, never sent to the hub |
| Host records          | runner | [host-records.md](./host-records.md)                                                                                 |
| Hub settings record   | hub    | [hub.md](./hub.md)                                                                                                   |

## Environment variables

An environment variable may supply a `boot` key, and where it does it overrides the file. A `pass`, `session`, or
`immutable` key is never read from the environment: a value fixed at process start would make a live setting a `boot`
one without saying so. The variables that supply a key, and those that are not settings, are listed in
[lifetimes.md](./lifetimes.md) §Environment variables. `BZ_RUNNER_TICK_SECONDS` is the one variable today that supplies
a setting whose lifetime is not `boot`, and it is deprecated (§The one new setting).

## Seeding from the file

At every startup, each daemon reconciles its file into its store one entry at a time. An environment variable never
seeds an entry, except the deprecated `BZ_RUNNER_TICK_SECONDS` until its removal:

- **The file states the entry and the store has none:** the entry is created with the file's value, its change-log row
  naming the seed as its writer.
- **The store already has the entry:** the file's value is ignored, with no warning. The store's value stands.
- **Neither has it:** the entry resolves to its registry default and nothing is written.

An entry is one key's whole value: a list, an alias table, or the throttle's `taper[]` array is one entry, never seeded
element by element. A per-runner key's entry is scoped to its runner, so a runner's entries are seeded from its
`[[runner]]` table the first time that runner starts, and a second runner's table never seeds the first's. The seed
writes through the same validation an API write does; an entry the file states but validation refuses fails the startup,
as the same value in the file fails a boot today.

## Runner

Five places freeze a runner's configuration at boot today, and each becomes an evaluation at the start of a pass:

| Frozen today                                      | Where                                                                                                                                                                                                      | Becomes                                                                                                                                 |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| The file, read once                               | `HostConfig.load`, `runner/host_config.py:567`                                                                                                                                                             | The `boot` keys still come from the file; every other key is resolved from the store's entries                                          |
| The loop's context, cached for the process's life | `RunnerStepper._context`, `runner/loop_wiring.py:429`; the runner prompt in `LoopWiring.of`, `:92`                                                                                                         | Rebuilt at the start of the first tick after the runner's settings revision moves                                                       |
| The tick interval, fixed into the timer thread    | `BZ_RUNNER_TICK_SECONDS`, `runner/cli/runtime.py:1117`; `PeriodicDriver`                                                                                                                                   | `tick_seconds`, evaluated by the driver before each wait                                                                                |
| Each runner's object graph                        | `runner/composition.py`: harness adapters and their alias tables, subscription samplers, trace settings, the worker telemetry plan                                                                         | Rebuilt per object when the revision of the settings it was built from moves, closing the one it replaces once nothing is mid-use on it |
| The set of runners                                | One `_RunnerThreads` per `[[runner]]` at boot (`runner/cli/runtime.py:1088`): the tick driver, the trace-sweep driver where tracing is on, and the runner's spawner executor (`runner/composition.py:873`) | Reconciled by the host tick (§The running set of runners)                                                                               |

How each becomes an evaluation is [passes.md](./passes.md).

### Host-level keys

| Key                                                                                                                              | `lifetime`         | Held today                                                                                                                                                                                                                                                                                                                                       |
| -------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `host`, `port`, `public_url`, `trusted_proxies`                                                                                  | `boot`             | The listener and the federation identity registered with the hub                                                                                                                                                                                                                                                                                 |
| `db_url`                                                                                                                         | `boot`             | The runner's store                                                                                                                                                                                                                                                                                                                               |
| `hub_url`, `host_token`, `session_secret_env`                                                                                    | `boot`             | Every hub client and the UI session signer, built at start                                                                                                                                                                                                                                                                                       |
| `hub_tokens`                                                                                                                     | `boot`, deprecated | Each runner's hub token by name; read only by the seed that imports it into the store ([runner-store.md](./runner-store.md) §Sealed tokens)                                                                                                                                                                                                      |
| `auth.hub_role_default`, `auth.superuser`, `auth.users`                                                                          | `boot`             | The runner's auth wiring                                                                                                                                                                                                                                                                                                                         |
| `loop`                                                                                                                           | `boot`             | Whether timer threads start at all (`runner/cli/runtime.py:1109`)                                                                                                                                                                                                                                                                                |
| `flags`                                                                                                                          | `boot`             | The flag registry, resolved at load                                                                                                                                                                                                                                                                                                              |
| `tools.env_passthrough`, `tools.path_prepend`                                                                                    | `session`          | The tool profile handed to each spawned process                                                                                                                                                                                                                                                                                                  |
| `tracing.platform`, `tracing.platform_sample_ratio`                                                                              | `boot`             | The platform tracer provider, installed once                                                                                                                                                                                                                                                                                                     |
| `tracing.harness_telemetry`, `tracing.worker_programs`, `tracing.worker_program_services`                                        | `session`          | The telemetry environment handed to a worker at launch                                                                                                                                                                                                                                                                                           |
| `tracing.sweep_seconds`, `tracing.settle_seconds`, `tracing.max_lag_seconds`, `tracing.batch_limit`, `tracing.replay_max_window` | `pass`             | The lease trace sweep, built once per runner                                                                                                                                                                                                                                                                                                     |
| `transcripts.ship`                                                                                                               | `pass`             | Whether the transcript drain runs                                                                                                                                                                                                                                                                                                                |
| `transcripts.record_max_bytes`, `transcripts.chunk_max_bytes`                                                                    | `pass`             | The transcript pump                                                                                                                                                                                                                                                                                                                              |
| `worker_stdout.retention_days`                                                                                                   | `pass`             | `LoopConfig`, read by the retention pass                                                                                                                                                                                                                                                                                                         |
| `workspace[].id`                                                                                                                 | `immutable`        | Operator-chosen, scoped to the host, with no default: a host has no workspace until one is declared or created ([runner-store.md](./runner-store.md) §No default workspace). Reported to the hub at each host tick (`wire/host.py:20`), declared in every runner binding, and recorded on every claim's route (`hub/domain/runners/route.py:23`) |
| `workspace[].root`, `workspace[].provider`, `workspace[].repo[]`                                                                 | `pass`             | Built into the workspace's environment provider at boot (`build_workspaces`, `runner/composition.py:411`; `runner/environments/factory.py`). The host tick rebuilds the provider when they move, and the next `acquire` uses it. Nothing is refused; the operator keeps the disk in step                                                         |
| `workspace[].envs`, `workspace[].max_environments`                                                                               | `pass`             | The environment pool and its cap, built into the provider; rebuilt by the host tick when they move                                                                                                                                                                                                                                               |
| `workspace[].base_branch`                                                                                                        | `pass`             | Built into the provider; reaches the next environment acquired                                                                                                                                                                                                                                                                                   |
| `workspace[].prompt`, `workspace[].prompt_file`, `workspace[].prompt_package`                                                    | `session`          | The workspace prompt, resolved once at start                                                                                                                                                                                                                                                                                                     |
| `subscription[].slug`, `subscription[].provider`                                                                                 | `immutable`        | The slug keys usage samples, usage-limit marks, every runner's `runner[].subscription`, and the hub's declaration (`host_report`, `runner/composition.py:825`); the provider selects the sampler and renewer, and the shape of the usage history they keep                                                                                       |
| `subscription[].name`                                                                                                            | `pass`             | A label, reported to the hub at each host tick                                                                                                                                                                                                                                                                                                   |
| `subscription[].credentials_path`, `subscription[].usage_api_base`                                                               | `pass`             | Built into the subscription's sampler and credential renewer at boot (`runner/composition.py:440`, `:466`); rebuilt by the host tick when they move. A worker that reads `credentials_path` reads it at launch, so for a running session it is `session`                                                                                         |
| `subscription[].sample_interval_seconds`                                                                                         | `pass`             | The sampler's and renewer's cadence within the host tick; rebuilt with them                                                                                                                                                                                                                                                                      |
| `tick_seconds` (new)                                                                                                             | `pass`             | —                                                                                                                                                                                                                                                                                                                                                |

### Per-runner keys (`runner[]`)

| Key                                                                               | `lifetime`  | Held today                                                                                                                                                                                                                                                                           |
| --------------------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                                                              | `immutable` | Minted by the hub at `runner add`; keys the runner's store rows and leases, and every worker's `BLIZZARD_RUNNER_URL` path (`runner/config.py:966`)                                                                                                                                   |
| `name`                                                                            | `pass`      | `LoopConfig.runner_name`, declared at every registration — a label, never the runner's identity (`runner/loop/context.py:65`). Nothing outside the store is keyed by it once tokens are sealed by id ([runner-store.md](./runner-store.md) §Sealed tokens), so a rename is one write |
| `subscription`                                                                    | `pass`      | `LoopConfig.subscription`, the slug of the declared subscription the runner spends                                                                                                                                                                                                   |
| `transcripts_root`                                                                | `session`   | Where the runner reads its harness transcripts from                                                                                                                                                                                                                                  |
| `projects[]`                                                                      | `pass`      | `LoopConfig.projects` and `bindings`; declared to the hub at the next registration                                                                                                                                                                                                   |
| `agents`                                                                          | `pass`      | `LoopConfig.agents`; lowering it stops claims until the running count is under it, and stops no session                                                                                                                                                                              |
| `gates`                                                                           | `pass`      | `LoopConfig.gates`                                                                                                                                                                                                                                                                   |
| `queue.strict`                                                                    | `pass`      | `LoopConfig`, read by the claim step                                                                                                                                                                                                                                                 |
| `throttle.budget.usd`, `throttle.budget.window_hours`                             | `pass`      | `LoopConfig.throttle`, read by the throttle at each claim decision                                                                                                                                                                                                                   |
| `throttle.pace.lead_pct`, `throttle.pace.window`, `throttle.pace.taper[]`         | `pass`      | As above                                                                                                                                                                                                                                                                             |
| `throttle.schedule.days`, `throttle.schedule.hours`                               | `pass`      | As above                                                                                                                                                                                                                                                                             |
| `throttle.utilization.below_pct`, `throttle.utilization.windows`                  | `pass`      | As above                                                                                                                                                                                                                                                                             |
| `throttle.on_decline`, `throttle.on_usage_limit`                                  | `pass`      | As above                                                                                                                                                                                                                                                                             |
| `cost.chunk_cap_usd`, `cost.runner_ceiling_usd`, `cost.window_hours`              | `pass`      | Read at each spend check                                                                                                                                                                                                                                                             |
| `context.sample_interval_seconds`                                                 | `pass`      | `LoopConfig`, the context-sample cadence                                                                                                                                                                                                                                             |
| `context.warn_tokens`                                                             | `pass`      | Read at each sample                                                                                                                                                                                                                                                                  |
| `models.aliases`, `models.supported`, `effort.aliases`                            | `session`   | The Claude Code adapter's resolution tables; `models.supported` is also advertised to the hub at the next registration                                                                                                                                                               |
| `opencode.models.aliases`, `opencode.models.supported`, `opencode.effort.aliases` | `session`   | The OpenCode adapter's resolution tables, as above                                                                                                                                                                                                                                   |
| `claude_code.enabled`, `opencode.enabled`                                         | `pass`      | Which harnesses the runner advertises; a disabled harness's running sessions finish                                                                                                                                                                                                  |
| `claude_code.binary`, `opencode.binary`                                           | `session`   | The binary a launch execs; re-probed for its version at the next health refresh                                                                                                                                                                                                      |
| `opencode.auth_path`, `opencode.worker_config_path`, `harness.config_dir`         | `session`   | Paths handed to a worker at launch                                                                                                                                                                                                                                                   |
| `harness.autonomy`                                                                | `session`   | The permission mode composed into a worker's settings                                                                                                                                                                                                                                |
| `worker.env_passthrough`, `worker.path_prepend`, `worker.shell`                   | `session`   | The worker environment built at launch                                                                                                                                                                                                                                               |
| `runner_prompt`, `runner_prompt_file`                                             | `session`   | Resolved by `LoopWiring.of` when the stepper is built                                                                                                                                                                                                                                |

## The running set of runners, workspaces, and subscriptions

Each is a record of the host, added or created and retired through its API and taken up at the next host tick:
[host-records.md](./host-records.md).

## Hub

The hub's operational settings move from `blizzard-hub.toml` into one hub-wide settings record, beside the tenants'
records rather than in any of them, read per pass and per request ([hub.md](./hub.md)). Its entries are seeded from the
file by §Seeding from the file. Every value below is bound into a long-lived object in `build_services` today
(`hub/composition.py:1451`), or read from the boot configuration per request.

| Key                                                                                                                       | `lifetime`         | Moves to the record | Held today                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------- | ------------------ | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| `host`, `port`, `public_url`, `trusted_proxies`, `db_url`                                                                 | `boot`             | no                  | The listener, the store, and the public identity                                                                |
| `auth.mode`, `auth.superuser`, `auth.oauth.provider[]`                                                                    | `boot`             | no                  | The auth wiring and OAuth clients                                                                               |
| `flags`                                                                                                                   | `boot`             | no                  | The flag registry                                                                                               |
| `follow_latest`                                                                                                           | `boot`             | no                  | Read at start                                                                                                   |
| `auth.login_throttle.enabled`, `.capacity`, `.refill_seconds`                                                             | `pass`             | yes                 | `IpThrottle` (`hub/composition.py:1686`)                                                                        |
| `action_pass_seconds`                                                                                                     | `pass`             | yes                 | `ActionPassDriver`'s `PassBound` (`hub/composition.py:1514`)                                                    |
| `annotation_interval_seconds`                                                                                             | `pass`             | yes                 | The annotation sweep's interval and `AnnotationReconciler.full_pass_floor` (`hub/composition.py:1710`)          |
| `sweeps.<name>.mode`                                                                                                      | `boot`             | no                  | Whether `lifespan` starts the sweep's timer task (`hub/app.py:200`)                                             |
| `close_forge_writes_enabled`                                                                                              | `pass`             | yes                 | Consulted wherever a closer is handed out                                                                       |
| `produces_mode`, `route_token_mode`                                                                                       | `pass`             | yes                 | Read where a step's produces and route token are judged                                                         |
| `transcripts.record_max_bytes`, `transcripts.chunk_budget_max_bytes`, `transcripts.runner_daily_rate_max_bytes`           | `pass`             | yes                 | The transcript ingest service (`hub/composition.py:1021`)                                                       |
| `tracing.platform`, `tracing.platform_sample_ratio`                                                                       | `boot`             | no                  | The platform tracer provider                                                                                    |
| `tracing.sweep_seconds`, `tracing.settle_seconds`, `tracing.max_lag_seconds`, `tracing.batch_limit`                       | `pass`             | yes                 | `TraceExportSweep` (`hub/domain/observability/tracing/sweep.py:60`)                                             |
| `tracing.replay_max_window`                                                                                               | `pass`             | yes                 | `TraceReplay`                                                                                                   |
| `egress.directory`, `egress.format`, `egress.file_paths`, `egress.path_key_env`                                           | `boot`             | no                  | The egress writer and the process token it mints at start                                                       |
| `egress.datasets`, `egress.extractor_versions`, `egress.batch_limit`, `egress.max_rows_per_file`, `egress.min_free_bytes` | `pass`             | yes                 | `EgressWriterSettings` (`hub/composition.py:1567`)                                                              |
| `egress.sweep_seconds`, `egress.settle_seconds`                                                                           | `pass`             | yes                 | `EgressSweep` (`hub/domain/observability/egress/sweep.py:103`)                                                  |
| `egress.backfill_max_window`                                                                                              | `pass`             | yes                 | The backfill command's bound                                                                                    |
| `work_source[]`                                                                                                           | `boot`, deprecated | no                  | Read at start only to refuse it and name its import; `replaced_by` the work-source records (`epic:live-config`) |

## Lifetimes for constants the sweep will promote

Neither epic waits for the other, and they land in either order. `epic:config/sweep` turns hard-coded constants into
settings; this epic decides when a setting is evaluated. The table below is not work this epic does: it is the lifetime
each of these constants will declare when the sweep promotes it, agreed here so the sweep does not have to decide it
again.

- **This epic first:** a constant the sweep promotes later arrives with its lifetime, and if that is `pass` or `session`
  it is an entry from the start, seeded from the file and changed through its daemon's API.
- **The sweep first:** a promoted constant arrives as a file key with its lifetime declared, and is read at boot until
  this epic's piece for its daemon lands, which makes it an entry like every other key of its lifetime.

The constants listed are the ones a run is most often tuned by, which is why they are named ahead of time; the sweep's
full list is its own.

| Constant                                                                          | Daemon | Where                                                                                                         | Value               | `lifetime` |
| --------------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------- | ------------------- | ---------- |
| `HEARTBEAT_STALENESS_THRESHOLD`                                                   | runner | `runner/leases/activity.py:30`                                                                                | 1 h                 | `pass`     |
| `ELICITATION_STALENESS_THRESHOLD`                                                 | runner | `runner/leases/elicitation.py:34`                                                                             | 15 min              | `pass`     |
| `DEFAULT_RETRIES_MAX`                                                             | runner | `runner/loop/context.py:57`                                                                                   | 2                   | `session`  |
| `DEFAULT_CHECK_TIMEOUT`                                                           | runner | `runner/lifecycle/judgement/check_runner.py:15`                                                               | 600 s               | `session`  |
| `SHUTDOWN_DRAIN_DEADLINE`                                                         | runner | `runner/lifecycle/shutdown_drain.py:21`                                                                       | 60 s                | `pass`     |
| overload backoff base, cap, limit                                                 | runner | `runner/leases/overload.py:35`                                                                                | 60 s, 900 s, 5      | `pass`     |
| `FRESH_SAMPLE_INTERVALS`, `FLOOR_UNCONFIRMED_GRACE`, `FLOOR_NO_RESET_FALLBACK`    | runner | `runner/throttle/levers.py:38`, `runner/throttle/floor.py:30`                                                 | 3, 1 h, 30 min      | `pass`     |
| `DRAIN_LIMIT`, `HOST_LANE_DRAIN_LIMIT`, `RETENTION_FLOOR`                         | runner | `runner/lifecycle/drain.py:40`, `runner/lifecycle/host_lane_drain.py:24`, `runner/loop/retention_floor.py:11` | 100, 100, 1 h       | `pass`     |
| provisioning timeouts                                                             | runner | `runner/environments/internal/`                                                                               | 120, 60, 60, 300 s  | `session`  |
| hub HTTP timeout for the loop                                                     | runner | `runner/loop_wiring.py:66`                                                                                    | 30 s                | `pass`     |
| store retention windows                                                           | runner | `runner/store/internal/`                                                                                      | 7 d, 8 d, 1 d, 1 d  | `pass`     |
| `EVENT_DERIVATION_INTERVAL_SECONDS`, `TENANT_TEARDOWN_INTERVAL_SECONDS`           | hub    | `hub/app.py:161`, `hub/app.py:167`                                                                            | 60 s, 30 s          | `pass`     |
| `CLOSE_DRAIN_INTERVAL_SECONDS`, `CLOSE_DRAIN_BACKOFF_BASE_SECONDS`                | hub    | `hub/app.py:164`, `hub/domain/work_items/closure.py:51`                                                       | 60 s, one setting   | `pass`     |
| flow driver `TICK_SECONDS`, `WAKE_FLOOR_SECONDS`                                  | hub    | `hub/delivery/flow_driver.py:30`                                                                              | 2 s, 1 s            | `pass`     |
| `STUCK_AFTER`, `DEFAULT_POLL_INTERVAL`, `DEFAULT_POLL_TIMEOUT`, `PARK_POLL_FLOOR` | hub    | `hub/domain/delivery/flows.py:42`                                                                             | 3, 30 s, 3 h, 2 min | `pass`     |
| `DEFAULT_BOUNCE_CAP`                                                              | hub    | `hub/domain/graph/model.py:155`                                                                               | 5                   | `session`  |
| forge attempt timeout and retry delays                                            | hub    | `hub/forge/internal/github_landing.py:55`                                                                     | 30 s, (1 s, 3 s)    | `pass`     |
| session `IDLE_TTL`, `ABSOLUTE_MAX_AGE`                                            | hub    | `hub/auth/service.py:96`                                                                                      | 24 h, 30 d          | `pass`     |

## Across both

A few settings only make sense beside a partner on the other daemon — the hub's offline cutoff against the runner's
tick, its usage staleness against the runner's sample interval, its transcript record cap against the runner's. This
epic leaves each daemon free to set its own side and does not check one against the other; a runner and a hub set out of
step simply behave as their settings say. Having the runner take its side of each from the hub's settings is left for
later. Pairs inside one daemon are checked at its own write, once both sides are settings: the runner's
`api/hub_proxy.py` retry ceiling below the worker's read timeout, the heartbeat retention window above
`HEARTBEAT_STALENESS_THRESHOLD`, and the close-drain interval equal to its backoff base.

## Pinned

Constants with no lifetime: they stay in code, are never promoted, and the sweep leaves them where they are:
cryptographic sizes, wire-format limits (`wire/release.py`, `wire/analytics.py`), slug and name lengths, GitHub's own
limits mirrored in `hub/forge/internal/github_landing.py`, the admitted harness version ranges
(`runner/harness/claude_code/health.py:20`, `runner/harness/opencode/compatibility/probe.py:111`), the Anthropic
credential renewal lead inside Claude Code's own refresh threshold, and the usage-limit retention window above one week.
