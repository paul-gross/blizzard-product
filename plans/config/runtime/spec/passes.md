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
holding `_tick_lock`, the stepper reads its reader's `revisions()` ([runner-store.md](./runner-store.md) §The reader):

- **Unchanged:** the context the last tick built is reused.
- **Moved:** the stepper resolves the new `RunnerConfig`, builds a new `LoopWiring` through `LoopWiring.of`
  (`runner/loop_wiring.py:92`, which resolves the runner prompt), builds a new context from it, and swaps it in. The
  context it replaces is closed after the swap and still under the lock, so its hub client is never closed under a
  running tick, which `close()` (`:416`) does not guard against today. The context holds settings as data; the objects
  it uses are not rebuilt with it (§The objects settings reach). The worker-scratch sweep `_context` runs today
  (`runner/loop_wiring.py:437`) runs once, at the daemon's start, and never on a rebuild, since a rebuild can race a
  spawn.

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
would be at boot — and since every write validates the runner's whole configuration
([runner-store.md](./runner-store.md) §A write), a runner left bound to a retired workspace has every later write
refused until one fixes the binding. The operator fixes it in the same write as whatever else they meant to change. A
runner naming a subscription that has since been retired runs as though it named none
([host-records.md](./host-records.md) §Workspaces and subscriptions). The domain's statement in `execution/hosts.md`
§Bindings changes to say both.

## The trace sweep

The lease trace sweep, built once per runner today, takes its settings — `tracing.sweep_seconds`, `settle_seconds`,
`max_lag_seconds`, `batch_limit`, `replay_max_window` — from the snapshot at the start of each sweep, and its driver's
interval is `tracing.sweep_seconds` as that snapshot states it.

## The host tick

At its start the host tick reads the host revision and every record revision, and before its existing work — reporting
to the hub, sampling subscription usage, renewing credentials, watching for retired runners — it reconciles:

1. **The set of runners** ([host-records.md](./host-records.md) §Runners): starts each hosted runner that has no thread
   group, and stops each one retired.
2. **The workspaces.** Every reader of the host's workspaces — each runner's process and loop context, the routes
   (`app.state.workspaces`, `runner/app.py:454`), the health probes over the spawn roots (`runner/composition.py:911`) —
   reads them through one `HostWorkspaces` holder at its use, never a `WorkspaceRegistry`
   (`runner/environments/workspaces.py:107`) it was handed at composition. When any active workspace record's revision
   moved, the host tick builds a new registry from the active records and sets it on the holder in one assignment. Each
   workspace whose record did not move keeps its provider; every workspace keeps its lock, carried by its
   `workspace_id`, so an acquisition through a replaced provider and one through its replacement still take the same
   lock, and no environment is handed to two claims (`bzh:environment-held-once`). A replaced provider holds nothing
   that needs closing (`runner/environments/internal/basic_provider.py:52`, `winter_provider.py:59`), and an acquisition
   already running under it finishes under it. An environment a claim holds stays held, since what is held is read from
   the store. The registry may be empty: a host with no active workspace serves no binding, and its runners claim
   nothing until one is created.
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

## The objects settings reach

The objects built from a runner's settings today are built once, at composition, and hold the values they were built
with (`runner/composition.py:885`): the harness registry and its adapters, the worker telemetry plan, the harness health
probes. Under this epic none of them holds a setting. Each takes the settings it needs as an argument at the call that
needs them — an adapter's launch takes the session's settings ([§Sessions](#sessions)), a health probe its binary — and
is built once and never rebuilt. The two that hold real state are cached by their input instead: the harness version
cache by binary path, so a moved `claude_code.binary` is probed at the next health refresh and an unmoved one is not,
and the loop's hub client by its timeout. An adapter that disabling a harness leaves unused stays built; the harness
simply stops being advertised.

## Sessions

A claim decided in a tick is launched by the runner's spawner from that tick's context. The launch builds one
`SessionSettings` from the snapshot — every `session` key a worker is handed: the harness settings, model and effort
aliases, autonomy, the harness config directory, the worker's environment, shell, and path, the runner and workspace
prompts, the tool profile, `transcripts_root`, the telemetry plan — and hands it to the adapter's launch, which composes
the worker's command, environment, and settings from it and from nothing else. Each spawn records the two revisions of
the snapshot it launched from on its `lease_spawns` row ([runner-store.md](./runner-store.md) §Tables), and the lease's
view (`wire/lease.py:18`) gains `host_settings_revision` and `runner_settings_revision` from its latest spawn, so the
local page and `blizzard runner status` can say which side of an edit a session fell on. A session resumed after a
restart, or respawned after a retry, is a new spawn: it launches with the settings in force when it relaunches, and its
row says so.

A session's transcript is read from the `transcripts_root` it launched with, recorded on its spawn row, so moving
`transcripts_root` loses no running session's transcript.

A worker never re-reads a setting. A `credentials_path` a running session read at launch is the one it keeps, whatever
the subscription's record says later.
