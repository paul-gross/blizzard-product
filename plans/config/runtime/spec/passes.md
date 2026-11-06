# Passes on the runner

How each of the runner's passes evaluates its settings, how the objects built from settings are rebuilt and closed, how
the loop's own interval becomes live, and what a session takes with it. The rule every pass follows is
[lifetimes.md](./lifetimes.md) §Evaluated at the start of a pass; this file applies it to each of the five places a
runner freezes its configuration today ([settings.md](./settings.md) §Runner).

## The passes

| Pass            | Lock today                                                       | Evaluates at its start                                                                                                                                                                                      |
| --------------- | ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A runner's tick | `RunnerStepper._tick_lock` (`runner/loop_wiring.py:400`)         | The runner's revision and the host revision; rebuilds the runner's `LoopWiring` and context when either moved                                                                                               |
| A trace sweep   | `RunnerStepper._trace_lock`                                      | The same revisions; rebuilds the sweep's settings when they moved                                                                                                                                           |
| The host tick   | `HostPassStepper`'s `StepLock` (`runner/loop/step_passes.py:53`) | The host revision and the revisions of every hosted runner, workspace, and subscription record; reconciles the set of runners and rebuilds each workspace provider, sampler, and renewer whose record moved |

A step inside a pass — the transcript drain, retention, the context sample — reads from its pass's snapshot and takes
none of its own. A stepped loop (`loop = "stepped"`, `bzh:steppable-loop`) evaluates the same way at the start of each
stepped pass, so a test that writes a setting and steps the tick sees the setting take hold in that step.

## A runner's tick

`RunnerStepper._context` (`runner/loop_wiring.py:429`) stops caching for the process's life. At the start of each tick,
holding `_tick_lock`, the stepper reads `revisions(runner_id)` ([runner-store.md](./runner-store.md) §The reader):

- **Unchanged:** the context the last tick built is reused.
- **Moved:** the stepper resolves the new `RunnerConfig`, builds a new `LoopWiring` through `LoopWiring.of`
  (`runner/loop_wiring.py:92`, which resolves the runner prompt), builds a new context from it, and swaps it in. The
  context it replaces is closed after the swap and still under the lock, so its hub client is never closed under a
  running tick, which `close()` (`:416`) does not guard against today. A long-lived cache whose inputs did not move —
  the harness version cache, the retention passes — is carried into the new context rather than rebuilt.

The tick then records the revisions it started from as the runner's `in_force` ([runner-api.md](./runner-api.md)
§Views), registers with the hub from the new context, and claims from it. What the runner declares at registration — its
name, agents, gates, projects, bindings, supported models, enabled harnesses — is the snapshot's, so the hub's queue
answers the runner's settings from the first tick after a write.

**Lowering a count stops nothing.** A runner whose `agents` falls below the sessions it is running stops claiming until
it is under the line. A harness disabled while it runs sessions stops being advertised, and its sessions finish. A
project removed from `projects[]` takes no further claim, and a chunk of it already held finishes under the binding it
was claimed through (`execution/hosts.md` §Bindings: the workspace is fixed for the chunk's tenure).

**A binding whose workspace is gone stops nothing either.** Today a binding naming a workspace the host does not declare
halts the runner before it registers. Under this epic, a runner whose binding names a workspace that has since been
retired keeps running its other bindings and takes no claim through that one; the hub, which no longer sees the
workspace in the host's report, flags the binding as it does today (`hub/domain/runners/bindings.py:42`). A write that
binds a project to a workspace the host never declared, or to a retired one, is still refused by validation, as the file
would be at boot. The domain's statement in `execution/hosts.md` §Bindings changes to say so.

## The trace sweep

The lease trace sweep, built once per runner today, takes its settings — `tracing.sweep_seconds`, `settle_seconds`,
`max_lag_seconds`, `batch_limit`, `replay_max_window` — from the snapshot at the start of each sweep, and its driver's
interval is `tracing.sweep_seconds` as that snapshot states it.

## The host tick

At its start the host tick reads the host revision and every record revision, and before its existing work — reporting
to the hub, sampling subscription usage, renewing credentials, watching for retired runners — it reconciles:

1. **The set of runners** ([host-records.md](./host-records.md) §Runners): starts each hosted runner that has no thread
   group, and stops each one retired.
2. **The workspaces.** A new `WorkspaceRegistry` (`runner/environments/workspaces.py:107`) is built from the active
   workspace records when any of their revisions moved, reusing the provider and the lock of every workspace whose
   record did not move, and swapped in under the registry's lock so no acquisition sees half of it. A workspace whose
   record moved gets a new provider; the one it replaces holds no resource that needs closing, since neither provider
   does (`runner/environments/internal/basic_provider.py:52`, `winter_provider.py:59`), and an acquisition already
   running under the old one finishes under it. An environment a claim holds stays held, since what is held is read from
   the store.
3. **The subscriptions.** The usage-sample pass and the credential-renewal pass rebuild the sampler and the renewer of
   each subscription whose record moved, from its record; their cadence state is already in the store
   (`external_usage_samples`, the renewal claims), so nothing is lost.
4. **The host report** (`host_report`, `runner/composition.py:824`) is built from the snapshot each tick instead of the
   `functools.partial` bound at composition (`:752`), so a created or retired workspace or subscription reaches the hub
   at that tick's report.

## The loop's interval

`PeriodicDriver` (`runner/loop_wiring.py:454`) and `PeriodicPassDriver` (`runner/loop/periodic_pass_driver.py:16`) take
an `interval: Callable[[], float]` in place of their fixed `interval`, called before each wait. A runner's tick driver
and the host tick driver read `tick_seconds` from the last snapshot their pass took; the trace sweep reads
`tracing.sweep_seconds`. `tick_seconds` is host-level: every runner on the host and the host tick share it.

A wait already begun would hold a shortened interval to the old one, so a committed write of `tick_seconds` or
`tracing.sweep_seconds` sets a wake event the drivers wait on; a woken driver evaluates the new interval, counts it from
the start of the pass before, and either runs at once, if that moment has passed, or waits out the rest. A write never
interrupts a pass that is running. Jitter is drawn as today.

## Sessions

A claim decided in a tick is launched by the runner's spawner from that tick's context, so every `session` key a worker
is handed — the harness settings, model and effort aliases, the worker's environment, shell, and path, the runner and
workspace prompts, the tool profile, the telemetry plan — is the snapshot of the tick that claimed. Each spawn records
the two revisions of the snapshot it launched from on its `lease_spawns` row ([runner-store.md](./runner-store.md)
§Tables), and the lease's view (`wire/lease.py:18`) gains `host_settings_revision` and `runner_settings_revision` from
its latest spawn, so the local page and `blizzard runner status` can say which side of an edit a session fell on. A
session resumed after a restart, or respawned after a retry, is a new spawn: it launches with the settings in force when
it relaunches, and its row says so.

A worker never re-reads a setting. A `credentials_path` a running session read at launch is the one it keeps, whatever
the subscription's record says later.
