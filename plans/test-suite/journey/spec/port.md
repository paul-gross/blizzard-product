# The port

## Topology

The rehearsal is one test in `tests/journey/`, a folder of the Playwright package outside `tests/service/`, in a project
of its own, `journey`, which `playwright.config.ts` declares by name. Its name keeps it out of every `service-*`
selection, so it never runs in the service tier's jobs or under `blizzard:service-test`. The upper tiers' `journey` job
selects it by name, under `full_suite`, as it runs the Python rehearsal today.

It runs on a fresh hub and a runner host of its own. Its graph uses only the shipped `land` action, so its hub needs no
test-action profile. Its subject is a night, so its runner loop is timed, as an operator's is, and its hub's sweeps are
timed too. It mints its world through the harness, two repositories on the mock forge as its work source, with work
sources, repositories, and their secrets seeded through the hub's API. It kills the runner with the harness's
`runnerHost.kill` and the hub with `freshHub.kill`, and restarts each on its directory and port: at the kill no worker
is running, since every chunk is latched in a state whose worker has exited, so a kill that takes the runner's process
group loses nothing the rehearsal relies on.

## The night

The steps keep the original's order and its waits on each chunk's latched state: five issues filed and ingested, two
grouped and one moved in the queue; the runner widened to four environments and two agents through the host file; the
grouped, review, ask, and escalation chunks driven to `done`, `done`, `waiting_on_human`, and `needs_human`; both
daemons killed and restarted; the chunks' states unchanged by the restart; the invariant check, after the runner's first
completed pass; the escalation's takeover command and its wrapped form compared exactly against the runner's name, then
run verbatim; the question answered through the `cli` fixture; and the morning.

## The morning

| Assertion                                                                                                     | Today                                     | After the port                                                                   |
| ------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------- |
| The grouped work landed once on both repositories' `main`                                                     | git on the bare origins                   | The harness's reads of the bare origin: tree, history by path, and file contents |
| The failed review was rebuilt; its files landed, the escalated chunk's did not                                | git                                       | The same reads                                                                   |
| Each chunk's history: the grouped edges, the review's fail and pass, the findings asset, the commit artifacts | `GET /api/chunks/{id}`                    | The same                                                                         |
| The ask resumed without a takeover; the takeover advanced its session                                         | the mock session's files                  | `runnerHost.mockSessions()`, the mock fleet's own record                         |
| The takeover command names the runner                                                                         | `load_runner_config`                      | `GET /api/host/runners`                                                          |
| No environment orphaned                                                                                       | only through `UniqueEnvBinding`           | `GET /api/host/environments`, holding the escalated chunk's alone                |
| Both stores keep their invariants, after the restart and in the morning                                       | `Invariants` in-process, and the CLI once | `blizzard dev check-invariants` in client mode ([check.md](./check.md))          |
| `hub status` tells the truth                                                                                  | the CLI                                   | The `cli` fixture                                                                |

The bare-origin reads are the fleet tier's harness gain, built there for its own ports; the rehearsal needs the history
by path and the file contents it adds. The invariant check needs one invocation to reach both daemons, so the `cli`
fixture, which carries the hub's URL and session, passes the runner host's directory beside them for this command.

The invariant check the original makes in the instant after the kill, with both daemons dead, is dropped: no daemon is
there to ask, the checks after the restart cover the same recovery, and the crash sweep proves that window in-process.

## What leaves and what stays

`tests/journey/`'s Python, its `conftest.py`, and the `journey` marker in `pyproject.toml` are deleted in the port's
change. `mise run journey` stays, re-pointed at the Playwright project, so `scripts/gate-full.sh`, `docs/ci.md`, the
`blizzard:journey` matrix row, and its command page keep their command; the command page's machine-checked command text
and test path change in the same change, and `tests/test_default_suite_markers.py` drops the marker. The method keeps
its name. `tests/crash/support.py` is untouched: the helpers the rehearsal borrowed stay there for the sweep, and the
port has its own in TypeScript.
