---
epic: test-suite/hub-tier
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/hub-tier`

Most of the hub's service tests already describe it from the outside: they send HTTP to a hub subprocess, drive the mock
runner and forge through their levers, and read the answers. What holds them in Python is the launcher around them and a
handful of shortcuts. This epic moves them to the foundation's Playwright package, into `tests/service/hub/`, onto a
shared hub wherever they can share, and builds the few surfaces their shortcuts stood in for.

## The tests it moves

| Group                      | Size     | What it needs before it can move                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| -------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hub over HTTP              | about 67 | Mostly the foundation. A few files import a constant or a wire model, which the generated wire types replace. `finding_exits` drives the real CLI, which runs as a subprocess and names its tenant with `--tenant`. `event_log` reads the hub event log, which belongs to no tenant, so it moves to `fresh-hub`. The two `platform_tracing` tests that drive only a hub join this group.                                                                                                |
| Hub with authentication on | about 11 | A person, without writing one into the store. Today `seed_invitation` inserts an invitation row. The black-box path boots with `auth.superuser`, signs that person in through the stub IdP, and invites through `POST /api/invitations`. The spike walked that sign-in end to end with no store access. Most of these tests vary the identity providers or depend on the superuser bootstrap, which happens once per hub, so most land on `fresh-hub` or on the authentication profile. |

### Signing in

A sign-in through the stub IdP is already fast: it shows no login page, and its authorize step redirects straight back
with a code, so the whole dance is a few redirects with no browser. What it cannot do is tell two sign-ins apart. It
holds one identity for the whole process and hands it to whichever sign-in completes next, so on a shared hub one
worker's sign-in would come back as another worker's person.

The IdP, in `blizzard-mock`, learns to hold many identities at once and to choose among them by the `login_hint` on its
authorize URL, the standard OIDC parameter, so it stays shaped like a real provider. A `signedIn(person)` fixture
registers the person with the IdP, starts at the hub's own `/api/auth/<provider>/authorize`, adds the hint as it follows
the redirect to the IdP, and returns a request context holding that person's session. The hub is untouched and every
sign-in walks the real flow. A member is invited through `POST /api/invitations` and then signs in the same way, and
since that person is the test's own, the test can stay at `common`; the superuser is declared in the authentication
profile and signed in once by the setup project. The epic confirms that many sign-ins on one hub stay under the login
throttle before the group ports. Tests whose subject is the superuser bootstrap or a change of identity providers still
take `fresh-hub`, because both are hub-wide.

Every test this epic moves lands in `tests/service/hub/`. The tests that start a real runner host beside the hub,
`runner_init` and `idp_federation`, belong in `tests/service/fleet/` and move with the runner tier, which publishes the
host layout they write today through internal classes.

## The tests that stay in Python

Some service tests have Python itself as their subject, and a TypeScript port would erase what they test. They stay in
pytest, in the service tier's pytest half, where the mock fleet is checked out: `opencode_compatibility`,
`harness_health`, `credential_renewal`, and `runner_selftest_mock` drive a Python adapter against a mock binary;
`wire_parity` and `parity_guard` hold the Python hub client to the wire and the mock; `mock_data_seeding`'s subject is
the mock-data tool writing into a hub's store; and `host_token`, `runner_bindings`, and `mock_chunk_findings` exercise
the mock hub itself. `tenant_teardown` builds a hub in-process and moves down to the component tier.

## Classifying before porting

The epic opens by classifying every file it moves, so the cost of the `fresh-hub` tests is known before the first port
rather than discovered at the last. Each test gets its isolation level, the loosest its reads allow under the
foundation's rules: `common`, `fresh-project`, `fresh-tenant`, or `fresh-hub`. The levels are also where the tier's
speed comes from, so a test is not placed at `fresh-tenant` by habit. Route-token and produces enforcement, the forge
annotation interval, and an authentication setup used by several tests are hub-wide startup settings and ride profiles.
A test whose subject is something multi-tenancy leaves global takes a fresh hub, as the foundation lists, and so does
any test that restarts the hub. The forge-status test asserting that no sweep loop starts is the model case: on a shared
hub it could only show that its own issue went unlabelled, which is a weaker claim, so it takes `fresh-hub` instead.

## Surfaces it builds

- **The tenant on every pass's log lines.** A per-tenant background pass binds its tenant to the lines it writes, as a
  tenant-scoped request already does, so a test on a shared hub can assert on them and an operator can read one tenant's
  activity out of a shared log.
- **Keyed sign-in** in the stub IdP, as above.
- **A cursor read** on the event stream, so a client can join from the tenant's current position, as below.
- **Whatever a ported test proves missing.** A test that cannot be written against today's surfaces either finds its
  state through the real flow or earns a surface an operator would want too. A test-only route behind a configuration
  flag is the last resort, and each one is named in this plan when it is added.

## Waits that end when the system has spoken

The hub's SSE tests wait on the clock today. `sse_tap` drains the connect-time replay for a fixed 1.5 to 2 seconds,
because the end of the replay cannot be detected, and `collect(window=6.0)` holds a fixed window open, because a
negative assertion such as "exactly one `queue-changed`" has no other way to show nothing more arrived. The ported tests
are written without either:

- **No replay to drain.** At `fresh-tenant`, a test opens its stream as soon as its tenant exists, before it acts, and
  since its tenant is new, the stream holds the tenant's whole history from its first event. On the common tenant the
  history is every other test's, so a test at `common` or `fresh-project` opens its stream from the tenant's current
  cursor and reads it filtered to its own ids. That needs a cursor read, which this epic builds; it serves any test that
  must join a stream late, and an operator's client resuming from now rather than from the beginning, in place of a
  sleep.
- **Positive assertions** return as soon as the expected events have arrived.
- **Negative assertions** perform a known, harmless act in the test's own world after the act under test and wait for
  the event it emits, an act on something only that test made. The stream is ordered, so anything the act under test
  emitted arrives first. This holds only where fan-out is emitted before the request returns, so each event path is
  verified, and one that fails keeps a bounded window with a comment saying why.

## Done

Every hub-side daemon test runs in `tests/service/hub/`. The tests whose subject is Python run in the service tier's
pytest half, `tenant_teardown` runs in the component tier, and the blizzard-context citations of every moved test name
its new home.
