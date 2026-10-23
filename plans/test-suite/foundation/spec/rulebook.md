# The rulebook companion

blizzard-context is what a fleet worker reads before writing a test, so it changes in a companion that lands with
[the package](./carry-over.md#the-order-of-landings), and the exemplar it points at lands with the ports. Paths in this
leaf are relative to the `blizzard-context` repo root. Every rule added follows the `Rule` / `Why` / `Detect` / `Do` /
`Don't` skeleton and earns its cold-spawn eval, as `CONTRIBUTING.md` requires.

## The tier, redefined

- **The matrix row.** `verification/blizzard.md`'s `blizzard:service-test` row names both halves:
  `mise run playwright -- --project='service-*'` for the Playwright package and `mise run service-test` for the pytest
  cases that stay.
- **The tier's detail.** `verification/blizzard/commands/test-tiers/service.md` states the tier as the two halves, what
  each holds, and that the pytest half holds only cases whose subject is Python itself, such as the external-CLI
  bindings `bzh:external-cli-fake-is-service-tier` keeps there. Its machine-parsed `### blizzard:service-test` section
  carries both command spans paired with their `mise run` tasks, so the registry check's command pairing holds for each.
- **The roster.** `verification/blizzard/tier-rules.md`'s **Service** row reads "Playwright Test + HTTP; pytest for an
  external-CLI binding" in its tooling column.
- **New daemon tests.** A rule in `tier-rules.md`, `bzh:service-tests-in-playwright`: a new service test whose subject
  is a daemon's behaviour is written in the Playwright package, and a pytest service case is added only under
  `bzh:external-cli-fake-is-service-tier`. Its `Detect` is a new `test_*_service.py` in `tests/service/` whose subject
  is not an external-CLI binding.
- **Ports move their citations.** A rule beside it, `bzh:port-moves-citations`: the change that ports a test deletes its
  Python original, re-points every citation of the original in blizzard-context and blizzard to the spec that replaces
  it, keeping the function name as the test's title, and drops the original's entries from
  `scripts/prose-density-baseline.json`. Its `Detect` is the registry check's existence check on a cited file.

## The rules scoped

- **The statement guards.** `verification/blizzard.md`'s paragraph naming "every `blizzard:` row that runs pytest" names
  instead every daemon any tier spawns, whatever runs the tier, and names `tests/daemon-env.json` as where the guards
  are switched on for both suites. The two guard sections in `tier-rules.md` say the same, and that the harness's
  harness fails a Playwright run on any refusal in a daemon's log, attributed to the test or tenant that caused it.
- **Test data.** `tier-rules.md`'s *Hermetic by construction* line, "Stand test data up through the mock-data CLI and
  its fixtures", binds the Python tiers only. A Playwright test builds its data through the hub's public surfaces and
  never through `tool:mock-data`, and `verification/blizzard/tools.md`'s `tool:mock-data` entry says so.
- **SQLite.** The rule that tests run against SQLite stands unless
  [the store measurement](./carry-over.md#the-measurements) moves the shared hubs to Postgres, in which case that change
  amends it.
- **The frontend toolchain.** `standards/frontend.md`'s `bzh:frontend-toolchain` scope already binds eslint without
  prettier across the whole workspace. It gains `blizzard/tests`'s TypeScript by name, and its single-test-runner half
  stays the unit and component tier's.
- **Comments.** `standards/comments.md`'s `bzh:comment-locality` scope table, which `bzh:comment-encapsulation` follows,
  gains a row binding `.ts` files under `blizzard/tests`, excluding `tests/harness/client/wire/`, which is generated, as
  `web/`'s generated client is excluded.
- **The pre-push sweep.** `verification/blizzard/pre-push.md`'s `bzh:sweep-release-only-tiers` grep already reaches
  `tests/service/`; its command gains `--exclude-dir=node_modules`, so the sweep reads the specs and not the package's
  dependencies.

## The registry check

`scripts/check-registry-drift.py` attributes a pytest file to a tier by collecting `tests/` under each tier's marker,
and cites only names matching `test_*.py`, so a Playwright spec is invisible to it. It learns specs as firmly:

- **Collecting.** It runs `npx playwright test --list --reporter=json` in the blizzard checkout's `tests/`, which loads
  every spec and lists every test's file, title, and project without starting a setup project or any daemon. A spec
  whose project starts `service-` is attributed to `blizzard:service-test`.
- **Citing.** Its citation pattern gains `tests/service/**/*.spec.ts` paths and bare `<name>.spec.ts` names, and the
  `file::title` form, the title being the ported function name.
- **Checking.** Check A holds a cited spec to existing, and a `file::title` citation to a title the listing holds. Check
  B2 holds a spec cited inside `### blizzard:service-test` to being attributed there. Check D2 pairs
  `mise run playwright` with its task's `run`.

The script's module docstring drops the limitation it now lifts, and its own tests,
`blizzard-context:registry-drift-tests`, gain a listing fixture covering each case.

## The tier's invariants

A fleet worker arrives knowing how Playwright tests are usually written, and most of that habit is wrong here:
Playwright's documentation and the examples an agent has absorbed reach for the built-in `request` fixture, write full
URLs, wait with `waitForTimeout`, and set up state in `beforeAll`, and each of those breaks a shared hub. A new spoke,
`verification/blizzard/service-tests.md`, routed from the tier's detail and from `exemplars/index.md`, states the
invariants as rules, each with the reason it exists:

| Rule                              | It says                                                                                                                                                                                                           |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bzh:service-fixtures-only`       | A test names the fixtures it needs, its tenant, project, clients, people, and event stream, builds none itself, imports only from `tests/harness/fixtures.ts`, and never uses Playwright's built-in `request`     |
| `bzh:service-clients-only`        | Every call goes through the harness's clients: no raw `fetch` against a daemon, no hand-built URL, and a route a test needs that a client lacks is added to the client rather than worked around                  |
| `bzh:service-isolation-earned`    | A test declares the loosest level its reads allow, and at `common` and `fresh-project` names what it creates uniquely, asserts only on its own ids, and asserts no count, emptiness, or absence across the tenant |
| `bzh:service-waits-on-the-system` | A test waits on an event or a state it can read, never on the clock, and a negative assertion waits on a barrier act only it would emit                                                                           |
| `bzh:service-public-surfaces`     | No store access outside a `fresh-hub` test whose subject is the store, against files before the daemon starts or after it stops, and no seeding through `tool:mock-data`                                          |
| `bzh:service-own-mock-state`      | A mock-fleet lever touches only the test's own repository, issue, or runner; never a lever with no `repo`, and never the forge's lever reset outside a fresh hub                                                  |
| `bzh:service-config-is-a-profile` | A hub-wide setting rides a profile or a fresh hub, never a test that changes it in place                                                                                                                          |

Each rule's `Detect` names what enforces it, where the harness does. The spoke closes by saying that the rules grow as
the ports teach them: a practice a later epic settles, which a capable agent would not arrive at on its own, joins the
spoke in the change that settles it.

## What the harness enforces

The package's lint, `tests/eslint.config.mjs`, enforces in specs what static analysis can see:

- `no-restricted-syntax` refuses destructuring `request` from a test's fixtures, and any call of `waitForTimeout`.
- `no-restricted-globals` refuses `fetch` in a spec.
- `no-restricted-imports` refuses any harness module but `harness/fixtures` in a spec.
- `no-restricted-syntax` refuses `test.beforeAll` and `test.afterAll` in a spec, since shared state across a file's
  tests belongs in a fixture.

At run time, the factory fixtures that would furnish the shared tenant refuse below `fresh-tenant`, and
`hubLog.untagged()` refuses a hub-wide log read below `fresh-hub`
([isolation.md](./isolation.md#the-clients-and-the-ledger)). The rest is the rules', and
[the cross-check](./isolation.md#the-cross-check) is what notices a test that breaks them.

## The exemplar

`exemplars/index.md`'s `## Tests` section gains a row for a service spec, pointing at the ported
`tests/service/hub/analytics-events.spec.ts` in `blizzard`, the port that exercises a tenant of its own, a runner token,
the event-derived reads, and a paged list. It lands with the ports, and until then the spoke stands without it.
`exemplars/conformance.md` (`bzh:exemplar-conformance`) places exemplars under `blizzard/exemplars/`, so it gains the
case of a live spec chosen as the reference, which stays where it runs and carries the `// exemplar:` comment naming the
rules it instantiates, since a copy outside the package could not run against a hub.
