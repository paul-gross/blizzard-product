---
epic: test-suite/hub-tier
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/hub-tier`

Most of the hub's service tests already describe it from the outside: they send HTTP to a hub subprocess, drive the mock
runner and forge through their levers, and read the answers. What holds them in Python is the launcher around them and a
handful of shortcuts. This epic moves them to the foundation's Playwright project, onto a shared hub wherever they can
share, and builds the few surfaces their shortcuts stood in for.

## The tests it moves

| Group                      | Size     | What it needs before it can move                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hub over HTTP              | about 67 | Mostly the foundation. A few files import a constant or a wire model, which the generated wire types replace. `finding_exits` drives the real CLI, which runs as a subprocess and names its tenant with `--tenant`. `event_log` reads the hub event log, which belongs to no tenant, so it moves to a fresh dedicated hub. The two `platform_tracing` tests that drive only a hub join this group.                                                        |
| Hub with authentication on | about 11 | A person, without writing one into the store. Today `seed_invitation` inserts an invitation row. The black-box path boots with `auth.superuser`, signs that person in through the stub IdP, and invites through `POST /api/invitations`. The spike walked that sign-in end to end with no store access. Most of these tests vary the identity providers or depend on the superuser bootstrap, which happens once per hub, so most land on dedicated hubs. |

The tests that start a real runner host, `runner_init` and `idp_federation`, move with the runner tier, which publishes
the host layout they write today through internal classes.

## The tests that stay in Python

Some service tests have Python itself as their subject, and a TypeScript port would erase what they test. They stay in
pytest, in the service tier's pytest half, where the mock fleet is checked out: `opencode_compatibility`,
`harness_health`, `credential_renewal`, and `runner_selftest_mock` drive a Python adapter against a mock binary;
`wire_parity` and `parity_guard` hold the Python hub client to the wire and the mock; `mock_data_seeding`'s subject is
the mock-data tool writing into a hub's store; and `host_token`, `runner_bindings`, and `mock_chunk_findings` exercise
the mock hub itself. `tenant_teardown` builds a hub in-process and moves down to the component tier.

## Classifying before porting

The epic opens by classifying every file it moves, so the cost of the dedicated hubs is known before the first port
rather than discovered at the last. Each test is shareable, on a profile, on a worker-scoped dedicated hub, or on a
fresh one. Route-token and produces enforcement, the forge annotation interval, and an authentication setup used by
several tests are hub-wide startup settings and ride profiles. A test whose subject is something multi-tenancy leaves
global takes a fresh hub, as the foundation lists, and so does any test that restarts the hub. The forge-status test
asserting that no sweep loop starts is the model case: on a shared hub it could only show that its own issue went
unlabelled, which is a weaker claim, so it goes dedicated instead.

## Surfaces it builds

- **Further vocabularies as data.** Where a ported test names a kind the foundation's fact-kind contract does not cover,
  hub event kinds first, that vocabulary joins it under `contracts/` the same way.
- **Whatever a ported test proves missing.** A test that cannot be written against today's surfaces either finds its
  state through the real flow or earns a surface an operator would want too. A test-only route behind a configuration
  flag is the last resort, and each one is named in this plan when it is added.

## Waits that end when the system has spoken

The hub's SSE tests wait on the clock today. `sse_tap` drains the connect-time replay for a fixed 1.5 to 2 seconds,
because the end of the replay cannot be detected, and `collect(window=6.0)` holds a fixed window open, because a
negative assertion such as "exactly one `queue-changed`" has no other way to show nothing more arrived. The ported tests
are written without either:

- **No replay to drain.** A test opens its stream as soon as its tenant exists, before it acts, and since its tenant is
  new, the stream holds the tenant's whole history from its first event. A test that must join later earns a surface for
  it, a cursor read or a frame marking the end of the replay, rather than a sleep.
- **Positive assertions** return as soon as the expected events have arrived.
- **Negative assertions** perform a known, harmless act in the test's own tenant after the act under test and wait for
  the event it emits. The stream is ordered, so anything the act under test emitted arrives first. This holds only where
  fan-out is emitted before the request returns, so each event path is verified, and one that fails keeps a bounded
  window with a comment saying why.

## Done

Every hub-side daemon test runs in the Playwright project. The tests whose subject is Python run in the service tier's
pytest half, `tenant_teardown` runs in the component tier, and the blizzard-context citations of every moved test name
its new home.
