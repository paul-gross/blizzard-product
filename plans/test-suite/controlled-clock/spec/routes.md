# Reading and moving a clock

Every daemon in the controlled mode serves the same pair of operations under its own path. A daemon on the system clock
serves neither, and answers `404` as for a route it never had.

## The shape

A read answers `200` with `{ "now": "<instant>" }`.

A move takes exactly one of two bodies:

- `{ "to": "<instant>" }` sets the clock to that instant.
- `{ "advance_seconds": <number> }` advances it by that many seconds, fractions allowed.

It answers `200` with `{ "previous": "<instant>", "now": "<instant>" }` once the clock reads the new instant, so every
read of the clock that starts after the answer sees it. Moving to the current reading, or advancing by zero, succeeds
and changes nothing. The refusals:

| Answer | When                                                                               |
| ------ | ---------------------------------------------------------------------------------- |
| `409`  | `to` is earlier than the current reading: `{ "code": "clock_backwards" }`          |
| `422`  | both bodies or neither, a negative `advance_seconds`, or an instant without a zone |

Instants are written as the hub writes every instant on its wire, `bzh:utc-instants`.

A move does nothing but move the clock. It runs no pass, wakes no sweep, and publishes no event. What the new instant
means is for the next pass the test steps.

## Where each daemon serves it

| Daemon                            | Read                   | Move                    | Who may call                                                                                                |
| --------------------------------- | ---------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| Hub                               | `GET /api/admin/clock` | `POST /api/admin/clock` | The hub-level `TENANT_ADMIN` permission, as `/api/admin/sweeps` requires (`src/blizzard/hub/api/sweeps.py`) |
| Runner host                       | `GET /api/host/clock`  | `POST /api/host/clock`  | The human lane, as `/api/host/ticks` is: free over `host.sock`, gated over TCP when the hub offers an IdP   |
| Mock hub, mock runner, mock forge | `GET /_clock`          | `POST /_clock`          | Anyone, as the mock fleet's other underscore routes allow (`blizzard_mock/forge/api/repos.py`'s `/_repos`)  |

One clock serves the whole runner host, so the route is host-wide rather than per runner: every runner on the host reads
the host's clock.

## The flow driver's step

A controlled hub starts no flow-driver timer. `POST /api/admin/test-flow/step` runs one flow-driver pass and answers
`200` once its writes are readable, with the clock's `now` and the pass's counts: `reconciled`, `skipped`, `failed`,
`submitted`, `closed`, `interrupted`, `started`, and `noticed`. Steps and clock moves serialize on one lock, so a move
never lands part-way through a pass. It takes the clock routes' permission and is mounted under the same fence, with
`include_in_schema=False`. A hub on the system clock keeps the driver's timer and answers `404`.

The route exists on the hub today, in `src/blizzard/hub/api/test_flow_control.py`, behind the flow-step control's own
switch. The epic keeps its path and answer and moves it behind the clock fence.

## Not a released surface

`bzh:released-surface-additive` holds every surface that `docs/versioning.md` names: the HTTP API, the wire,
configuration, the store, and the trace and egress contracts. `blizzard:compat` checks those surfaces through their
committed documents. The clock routes stay outside every one of them. They are mounted only when the fence permits the
controlled clock, and they are declared with `include_in_schema=False`, so they reach neither `openapi/hub.openapi.json`
nor `openapi/runner.openapi.json`. The switch is environment-only, so it never reaches `contracts/config/`.
`docs/versioning.md` gains one sentence stating the rule: a route or variable that exists only behind a test fence is
not part of any supported surface and may change in any release. The routes therefore need no deprecation, and the epic
carries no follow-up removal. Every landing adds behaviour that only a fenced daemon exposes, which keeps the epic
additive under the backwards-compatible rule every epic is planned to.

The harness's TypeScript clients are generated from those OpenAPI documents, so the clock fixture calls the routes
through a small hand-written client of its own rather than a generated one.

## Proof

The service tier proves each route on a fenced daemon: a read, a set, an advance, a refused backwards set, and the
effect a move has on a stepped pass. On the hub, the forge-status floor is the first such effect, and on the runner
host, a budget window ageing out. Each daemon's `404` without the fence is proved in the component tier, where building
an app with and without the fence costs nothing.
