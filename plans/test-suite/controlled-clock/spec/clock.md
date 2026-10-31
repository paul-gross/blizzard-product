# The controlled clock

## The clock

`src/blizzard/foundation/clock.py` gains `ControlledClock`, an `IClock` whose reading is an anchor instant moved only by
`set(to)` and `advance(by)`. It anchors at the machine's UTC time when it is built, and stands still from there. `set`
refuses an instant earlier than the current reading. `advance` refuses a negative duration. Both apply under a lock, so
concurrent readers see either the old reading or the new one, never a partial change. It differs from `FixedClock`,
which stays the component tier's in-process double, in two ways: it is thread-safe, and it refuses to move backwards.

A controlled clock is an `IClock` only. No `IMonotonicClock` reader is domain time, so none moves under the controlled
clock:

| Reader                                                                                                                             | Why it stays real                                                               |
| ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `Sweep.timer` (`src/blizzard/hub/app.py`) and `FlowDriver`'s timer (`src/blizzard/hub/delivery/flow_driver.py`)                    | They time a pass and space the next one. A stepped sweep starts no timer at all |
| `hub_retry_clock` behind `HubProxy` (`src/blizzard/runner/api/hub_proxy.py`)                                                       | A forward's retry backoff within one request, a mechanical deadline             |
| The subprocess deadlines in `src/blizzard/runner/process/` and `src/blizzard/runner/harness/opencode/compatibility/`, and selftest | Each waits on a real process                                                    |
| Span timestamps (`src/blizzard/foundation/span_clock.py`)                                                                          | Telemetry observes. A span stamped on a frozen clock would have no duration     |

The flow driver's reconcile floor, `RECONCILE_FLOOR`, is judged against `IClock`, so it moves with the controlled clock
even though the driver's two-second wake does not.

## The fence

Two environment variables, read once at boot:

- `BLIZZARD_CLOCK=controlled` asks for the controlled clock. Any other value, or none, is the system clock.
- `BLIZZARD_CLOCK_FENCE=1` permits it. Only the value `1` permits it, as only `1` permits `BLIZZARD_CRASH_FENCE`
  (`src/blizzard/foundation/crash.py`).

A daemon asked for the controlled clock without the fence runs on the system clock and logs one warning naming both
variables. A stray variable on a production host therefore changes nothing, which is the crash fence's own rule. The
harness proves the mode took hold by reading the clock route ([harness.md](./harness.md#aligning-a-stack)), so a fence
it forgot fails the test at once rather than leaving it on real time. Neither variable is a configuration key, so
neither reaches `contracts/config/`.

One function in `src/blizzard/foundation/clock.py` resolves the environment to a clock. Each composition root calls it
in place of `SystemClock()`.

## The composition roots

Only the serving processes take the controlled clock:

- **The hub's `host` process.** `build_hosted_app` resolves the clock once. It passes it to the core, which today
  `build_process_core` builds on `SystemClock()`, and to `_runtime_of`'s boot stamps, so the events recorded at boot are
  stamped on the same timeline. The offline commands that also call `build_process_core`, `rotate-key` and
  `import-legacy`, stay on the system clock: they run against a stopped hub, and no test moves their time.
- **The runner's `host` process.** `_build_host_graph` resolves the clock in place of its `SystemClock()`, and every
  runner on the host shares it through `host.clock`, the local API included. The runner's one-shot commands (`init`,
  `add`, `store reset`, `tick` without a running host) stay on the system clock. `blizzard-runner tick` against a
  running host steps that host, so it reads the host's clock.
- **The mock fleet.** The mock hub's, mock runner's, and mock forge's `create_app` resolve the same pair of variables
  through the same rule, implemented in `blizzard-mock` against its own clock protocol, since `blizzard-mock` does not
  import `blizzard`.

## The token check that must reach the clock

The runner judges a federation token's `exp` against its injected clock, but passes the token through PyJWT, which still
checks `iat` against the machine's clock with a thirty-second leeway (`FederationToken.identity`,
`src/blizzard/runner/auth/validate.py`). Once the hub's clock runs more than thirty seconds ahead of the machine's,
every token it mints would be refused as not yet valid. The check moves to the model: PyJWT is told not to verify `iat`,
and `FederatedIdentity.from_claims` refuses an `iat` later than the injected clock's `now` plus the same leeway. The
change is correct for production on its own terms, since every time check then reads the one injected clock, and it
lands before anything moves a clock ([carry-over.md](./carry-over.md#the-order-of-landings)).

The hub's OIDC exchange (`src/blizzard/hub/auth/oauth/internal/oidc_provider.py`) validates an identity provider's
`id_token` against the machine's clock, and the stub IdP (`blizzard_mock/idp/domain/signing.py`) signs from the
machine's clock. Both stay real. The exchange is a single moment between two parties on the same timeline, and what it
produces, the hub's session, then lives on the hub's clock.

## Proof

The component tier proves `ControlledClock`'s refusals and its readers' consistency under concurrent moves, the fence's
resolution in each case (asked and fenced, asked and unfenced, not asked), and the federation token's `iat` refusal on
an injected clock. The service tier proves the mode end to end through the routes ([routes.md](./routes.md#proof)).
