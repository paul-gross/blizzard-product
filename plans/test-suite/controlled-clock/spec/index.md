# Controlled-clock specifications

The technical contracts behind `epic:test-suite/controlled-clock`. Product intent begins at the
[epic plan](../index.md). The package, the harness, and the isolation levels belong to `epic:test-suite/foundation`
([its spec](../../foundation/spec/index.md)). The hub's stepped sweeps and a fresh hub's lifecycle belong to
`epic:test-suite/hub-tier` ([its spec](../../hub-tier/spec/index.md)), and the stepped runner host to
`epic:test-suite/runner-tier` ([its spec](../../runner-tier/spec/index.md)). Nothing here redefines them.

It builds on time as the daemons keep it today. `src/blizzard/foundation/clock.py` declares the two seams, `IClock`
(`now()`) and `IMonotonicClock` (`monotonic()` and `sleep()`), and their production and test implementations:
`SystemClock`, `FixedClock`, `SystemMonotonicClock`, and `ManualMonotonicClock`. `bzh:injected-clock` already holds
every instant in loop, store, and domain code to `IClock`. Each daemon builds its `SystemClock` once at its composition
root. For the hub, that is `build_process_core` in `src/blizzard/hub/composition.py`, which `build_hosted_app` in
`src/blizzard/hub/app.py` calls; the offline `init` and `migrate` commands stamp through `_runtime_of` in
`src/blizzard/hub/runtime.py`. For the runner host, it is `_build_host_graph` in `src/blizzard/runner/composition.py`,
which both `build_host_process` and the one-shot `build_runner_process` call, and whose `host.clock` reaches the local
API through `_wire_hosted_app` in `src/blizzard/runner/app.py`. The mock fleet keeps the same seam in
`blizzard_mock/clock.py`, used by the mock hub and mock runner, and in `blizzard_mock/forge/domain/clock.py`, and binds
a `SystemClock` in each app's `create_app`.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise. The epic reaches `blizzard-mock` for the
mock fleet's clocks, and `blizzard-context` for the rules it changes
([carry-over.md](./carry-over.md#citations-and-the-rulebook)).

| Where                            | Read when                                                                                                                                        |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| [clock.md](./clock.md)           | Building the controlled clock: the fence, the composition roots, what reads it and what stays real, and the token check it must reach            |
| [routes.md](./routes.md)         | Building the routes that read and move a daemon's clock, on the hub, the runner host, and the mock fleet, and why none is a released surface     |
| [harness.md](./harness.md)       | Building the clock fixture: who may control a clock, aligning a stack, moving it, the browser, and the check against reading the machine's clock |
| [ports.md](./ports.md)           | Moving a spec onto the controlled clock, or deciding whether a spec that reads time belongs to this epic                                         |
| [carry-over.md](./carry-over.md) | Landing the epic: the order of changes, the rulebook, and the open questions                                                                     |
