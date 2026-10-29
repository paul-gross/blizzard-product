# The port

## Topology

The rehearsal is one test in `tests/service/fleet/journey/`, its own project, `service-fleet-journey`, on a fresh hub
and a runner host of its own; the hub takes the fleet tier's `actions` profile if the rehearsal's graph uses a test
action, as its Python original's hub admits them. Its subject is a night, so its runner loop is timed, as an operator's
is, and its hub's sweeps are timed too. The upper tiers' `journey` job runs the project under `full_suite` in place of
`mise run journey`, so it stays outside `service-fleet`'s time. It mints its world through the harness, two repositories
on the mock forge as its work source, with work sources, repositories, and their secrets seeded through the hub's API.

The harness's `runnerHost.kill` sends SIGKILL to the host's process group, workers included. A daemon killed in the
night leaves its workers running, and the rehearsal's promise that work resumed without a takeover rests on that, so the
harness gains a kill of the daemon alone, which the port uses for both daemons.

## The night

The steps keep the original's order and its waits on each chunk's latched state: five issues filed and ingested, two
grouped and one moved in the queue; the runner widened to four environments and two agents through the host file; the
grouped, review, ask, and escalation chunks driven to `done`, `done`, `waiting_on_human`, and `needs_human`; both
daemons killed and restarted on their directories and ports; the escalation taken over with its command run verbatim;
the question answered through the `cli` fixture; and the morning.

## The morning

| Assertion                                                                                                     | Today                                     | After the port                                                          |
| ------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | ----------------------------------------------------------------------- |
| The grouped work landed once on both repositories' `main`                                                     | git on the bare origins                   | The same, through the harness's origin reads                            |
| The failed review was rebuilt; its files landed, the escalated chunk's did not                                | git                                       | The same                                                                |
| Each chunk's history: the grouped edges, the review's fail and pass, the findings asset, the commit artifacts | `GET /api/chunks/{id}`                    | The same                                                                |
| The ask resumed without a takeover; the takeover advanced its session                                         | the mock session's files                  | `runnerHost.mockSessions()`, the mock fleet's own record                |
| The takeover command names the runner                                                                         | `load_runner_config`                      | `GET /api/host/runners`                                                 |
| No environment orphaned                                                                                       | only through `UniqueEnvBinding`           | `GET /api/host/environments`, holding the escalated chunk's alone       |
| Both stores keep their invariants, after the restart and in the morning                                       | `Invariants` in-process, and the CLI once | `blizzard dev check-invariants` in client mode ([check.md](./check.md)) |
| `hub status` tells the truth                                                                                  | the CLI                                   | The `cli` fixture                                                       |

The invariant check the original makes in the instant after the kill, with both daemons dead, is dropped: no daemon is
there to ask, and the checks after the restart cover the same recovery.

## What leaves

`tests/journey/` is deleted in the port's change, with `mise.toml`'s `journey` task and the `journey` marker in
`pyproject.toml`. The method keeps its name, `blizzard:journey`, and its command page in blizzard-context names the
Playwright project. `tests/crash/support.py` is untouched: the helpers the rehearsal borrowed stay there for the sweep,
and the port has its own in TypeScript. `tests/journey/conftest.py`'s note that the rehearsal needs a local winter
source leaves with it.
