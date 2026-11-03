# The specs it moves

Each spec moves in a change of its own, keeps its title, and gives up its real-time wait or its read of the machine's
clock in that change.

## Specs that wait out a real window

| Spec                                        | The wait today                                                                                                                                                                                                                                                                                                                                                        | On the controlled clock                                                                                                                                                                                                                         |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tests/service/runner/budget-lever.spec.ts` | A trailing window shrunk to thirty seconds (`window_hours` of `30 / 3600`), and a poll of the runner's throttle read with a sixty-second deadline until the spend ages out                                                                                                                                                                                            | The window returns to whole hours. The test advances past it, steps the host, and reads the budget admitting                                                                                                                                    |
| `tests/service/fleet/forge-status.spec.ts`  | `annotation_interval_seconds = 1`, and two polls against a deadline of the five-second full-pass floor plus twenty-five seconds                                                                                                                                                                                                                                       | The interval returns to its default. The test advances past the floor, which `src/blizzard/hub/domain/observability/forge_status.py` judges against `IClock`, and steps the sweep                                                               |
| `tests/service/fleet/egress-night.spec.ts`  | A worker held fifteen real seconds at a checkpoint so its station is the night's slowest; a 1.1-second wait so the window closes on a whole second past every fact; `since` from `Date.now()`                                                                                                                                                                         | The worker stays held while the test advances the stack, so the station's recorded duration is set by the test rather than by real time. The window's bounds come from `clock.now()`                                                            |
| `tests/service/fleet/park.spec.ts`          | Two cases held under `test.fixme`, since each waits out a real hour: `test_park_timeout_then_person_merge_is_adopted_on_requeue` polls up to sixty-seven minutes for a one-hour `park_timeout_hours` to escalate `park-timeout`, and `test_second_parked_repository_has_its_own_review_clock` polls until the first repository's park marker is more than an hour old | The `test.fixme` comes off both. Each test advances past the hour, which `src/blizzard/hub/delivery/actions/landing/park.py` judges as `visit.now` against the park marker's `produced_at`, and reads what the deliver node's next poll records |

## Specs that build instants from the machine's clock

These specs do not wait, but they hand daemons instants taken from `Date.now()`, which the daemons judge against their
own clocks. That works only while both read the same machine at nearly the same moment. A reset aligned to the minute,
`minuteAhead`, can fall on either side of a minute boundary. On a controlled stack, the instants come from `clock.now()`
and the daemons agree with them exactly.

| Spec                                              | What it reads                                                                                               |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `tests/service/runner/subscription-floor.spec.ts` | `minuteAhead` for the resets a usage limit names                                                            |
| `tests/service/runner/release.spec.ts`            | `sample` and `minuteAhead` for usage windows and resets                                                     |
| `tests/service/runner/utilization-lever.spec.ts`  | `sample` for usage windows                                                                                  |
| `tests/harness/vendor-usage.ts`                   | `defaultLevels()` for every minted credential's windows ([harness.md](./harness.md#the-vendor-counterpart)) |

The egress night reads its exports through the published recipes, whose seven-day bounds are judged by the query
engine's own `now()` on the machine's clock. A step that ends in the stack's future still falls inside
`ended_at >= now() - INTERVAL 7 DAY`, so an advance of minutes leaves every recipe's answer unchanged.

The subscription floor's spec proves today that the floor opens early when the window turns over beneath its limit,
because waiting for the reset itself is out of reach. On the controlled clock, its port also proves the floor opening at
its reset: the test advances to the named instant, steps, and reads the floor admitting. It is a new case, added in the
port's change.

## Specs that read time for other reasons

These reach the check against the machine's clock ([harness.md](./harness.md#the-check-against-the-machines-clock)) but
not the controlled clock:

- `tests/service/hub/hub.spec.ts`'s `wholeSecondNow` builds sample instants on the shared hub. It reads `clock.now()` on
  a system stack, so nothing about it changes but its source.
- `tests/service/hub/hub.spec.ts`'s prompt-shutdown case and `tests/service/runner/runner.spec.ts`'s two timed shutdown
  cases have real elapsed time as their subject. Each keeps its read with a reasoned disable.
- `tests/service/fleet/collector-sampling.spec.ts` waits out the OpenTelemetry collector's own decision wait, a third
  party's real time, and keeps it the same way.

## What is not this epic's

`tests/service/test_credential_renewal_service.py` races two real `mock-codex app-server` processes refreshing one file.
It has no window to wait out, and it stays in the pytest half as `bzh:external-cli-fake-is-service-tier` places it. The
OpenCode spec's `JUDGE_WAIT_SECONDS` is a wait inside a worker's own program, not in the spec. The crash sweep and the
journey keep their own timing: the crash sweep stays in Python on purpose, and the journey may adopt the clock once both
have landed, under its own plan.
