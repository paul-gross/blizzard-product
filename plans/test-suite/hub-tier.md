---
epic: test-suite/hub-tier
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/hub-tier`

Most of the hub's service tests already describe it from the outside: they send HTTP to a hub subprocess, drive the mock
runner and forge through their levers, and read the answers. What holds them in Python is the launcher around them and a
handful of shortcuts. This epic moves them to the foundation's Playwright project, onto the shared hub wherever they can
share, and builds the few surfaces their shortcuts stood in for.

## The tests it moves

| Group                      | Size     | What it needs before it can move                                                                                                                                                                                                                                                                                                                       |
| -------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Hub over HTTP              | about 73 | Little beyond the foundation. A few files import a constant or a wire model, which a TypeScript test restates. `event_log` reads the hub event log through the store, and `GET /api/admin/events` serves the same rows. `finding_exits` drives the real CLI, which runs as a subprocess and names its tenant with `--tenant`.                          |
| Hub with authentication on | about 14 | A person, without writing one into the store. Today `seed_invitation` inserts an invitation row. The black-box path boots with `auth.superuser`, signs that person in through the stub IdP, and invites through `POST /api/invitations`, once per worker on the authentication profile. The spike walked that sign-in end to end with no store access. |
| No daemon at all           | about 47 | Nothing: these call Python adapters directly against mock binaries, compare Python protocols with the mock's routes, or run a hub in-process. They test the Python and stay in pytest, re-tiered out of the service tier.                                                                                                                              |

The first two groups move file by file, simplest first. Each test is classified as it moves: shareable, on a profile, or
dedicated. Route-token and produces enforcement, the forge annotation interval, and authentication are hub-wide startup
settings and ride profiles. Tracing configured through the environment, and any test that restarts the hub, keep a
dedicated one.

## Surfaces it builds

- **Fact kinds as data.** The `wire/facts.py` vocabulary is committed under `contracts/` with a drift test like the one
  for `contracts/cli/`. Tests that push facts over `POST /api/fleet/events` name kinds from it, and the `blizzard-mock`
  parity guard reads the same artifact instead of the Python source.
- **Whatever a ported test proves missing.** A test that cannot be written against today's surfaces either finds its
  state through the real flow or earns a surface an operator would want too. A test-only route behind a configuration
  flag is the last resort, and each one is named in this plan when it is added.

## Waits that end when the system has spoken

The hub's SSE tests wait on the clock today. `sse_tap` drains the connect-time replay for a fixed 1.5 to 2 seconds,
because the end of the replay cannot be detected, and `collect(window=6.0)` holds a fixed window open, because a
negative assertion such as "exactly one `queue-changed`" has no other way to show nothing more arrived. The ported tests
are written without either:

- **Replay.** Events carry integer ids, counted per tenant from one, so a test reads the current cursor and subscribes
  with `Last-Event-ID`, or drains until that id. It takes its position before it acts, because a frame can arrive before
  the request that caused it returns.
- **Positive assertions** return as soon as the expected events have arrived.
- **Negative assertions** trigger a known, harmless barrier event after the act and wait for it. The stream is ordered,
  so anything the act emitted arrives first. This holds only where fan-out is emitted before the request returns, so
  each event path is verified, and one that fails keeps a bounded window with a comment saying why.

## Done

Every hub-side service test runs in the Playwright project, the no-daemon tests run in the pytest tier they belong to,
and none of their Python originals remain under `tests/service/`.
