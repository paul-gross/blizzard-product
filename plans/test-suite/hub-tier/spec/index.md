# Hub-tier specifications

The technical contracts behind `epic:test-suite/hub-tier`. Product intent begins at the [epic plan](../index.md); the
package, the shared hubs, the isolation levels, and the clients it ports onto are `epic:test-suite/foundation`'s
([its spec](../../foundation/spec/index.md)), and nothing here redefines them. Implementation enters at the concern
below.

It builds on the hub's service tests as they run today. About sixty-five tests in `tests/service/` drive a hub
subprocess over HTTP, each starting its own through `tests/fleet_launchers.py`'s `run_hub` beside a mock forge, and most
of them a mock runner too. Eleven more drive a hub with authentication on, through launchers of their own and the stub
IdP, and give it its people by writing invitation rows into its store with `tests/support.py`'s `seed_invitation`.
Hub-wide settings ride `run_hub`'s keyword arguments: `route_token_mode`, `produces_mode`, `annotate` and
`annotation_interval_seconds`, `extra_env`. The event-stream tests read through `tests/service/support.py`'s `sse_tap`,
which drains the connect-time replay for a fixed `settle`, and `SseTap.collect`, which holds a fixed `window` open.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise. The epic reaches `blizzard-mock`, whose
stub IdP learns keyed sign-in ([surfaces.md](./surfaces.md#keyed-sign-in)), and `blizzard-context`, whose citations of
each moved test move with it ([carry-over.md](./carry-over.md#citations)).

| Where                            | Read when                                                                                                                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [roster.md](./roster.md)         | Placing a file — which tests move, at which level and on which profile, which stay in Python, which move down a tier, and which wait for the runner                                        |
| [surfaces.md](./surfaces.md)     | Building what the ports need of the product and the mock fleet — tenant tags on background passes, sweeps on request, the stream's join from now, the login throttle's keys, keyed sign-in |
| [harness.md](./harness.md)       | Extending the foundation's harness — people and sign-in, the CLI, profile placeholders, a fresh hub's own configuration and lifecycle, the OTLP sink                                       |
| [waits.md](./waits.md)           | Rewriting a test that waits on the clock — opening the stream, positive assertions, the barrier act, and the event paths it holds for                                                      |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of changes, the helpers that move first, the citations, and what is deleted                                                                                   |
