---
epic: test-suite/fleet-tier
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/fleet-tier`

Today's e2e tier is misnamed. Every scenario in it runs against the mock forge and mock harnesses, never a real work
source or a real GitHub repository, so it is the hub and a runner host held real with mocks around them, which is what
the initiative calls a fleet service test. Its browser scenarios are UI tests: the built board in a browser, driving a
real hub. Each one also starts a forge and a hub of its own, and the tier runs them one at a time, so a full run of its
84 cases costs about thirty-five minutes on a hosted machine. This epic moves each scenario to where its kind lives in
the Playwright package, and retires the e2e tier until blizzard can stand up a real work source under test.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

It follows the runner tier because it needs what that epic builds: a `blizzard-runner host` that steps on request and
serves its own local API, configuration as documented files, reads for the runner state its tests check, and the
harness's runner host bound to a real hub.

## The tests it moves

| Coupling today                                         | What replaces it                                                                                  |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| The runner loop ticked in-process, in most files       | the runner tier's stepping host, stepped through its local API                                    |
| The runner API served on a thread                      | the same host serving that API itself                                                             |
| Configuration built from Python objects                | the documented configuration files, and hub profiles the harness validates                        |
| The harness environment set on `os.environ`            | the environment of the host process                                                               |
| Hub and runner stores read while the daemons run       | a public read of each value, built here where the runner tier left none                           |
| Internal machinery imported as an oracle               | the behaviour those modules produce, observed through public surfaces, or a move down             |
| Test-only graph actions on a hub of the test's own     | a shared hub profile that admits them                                                             |
| A real OpenTelemetry collector beside each traced test | the harness's OTLP sink, decoding whole spans, and a real collector where sampling is the subject |

## Decisions

**Trace subtests become companion tests.** About nine files assert a fleet trace beside the behaviour they are about.
Hub tracing and its export endpoint are hub-wide, so a test that asserts on them needs a hub of its own, while the
behaviour beneath does not. Each such file splits: its core cases run at `fresh-tenant` on a shared hub, and its trace
assertions move to a companion test at `fresh-hub`, so the expensive isolation is paid only where tracing is the
subject.

**Test actions get a profile, not a rewrite.** Six files drive graphs through the hub's test-only actions, which a
shared hub refuses. Those tests run on a shared hub profile that sets the fence admitting them, the same fence their
Python originals set on a hub of their own, rather than having their graphs rewritten onto shipped actions they were not
written to test.

**Every missing read is built as a general read.** Where a test reads a value no public surface serves, the value gains
one an operator could use as well: flows, a graph's definition, a chunk's leases unbounded by the recent-lease cap, a
lease's resolved model, effort, harness version, and declared commits. Two needs are internal machinery instead, and
move down to the component tier: a runner's resume intents, and the forge client's conditional-read economy that one
test proves by scraping the mock forge's log. No surface returns a secret: the leak scans that read route tokens out of
the runner's store read them as the worker received them.

**Store carry-overs are dropped, not ported.** Three tests seed or read a stopped hub's store to prove a migration
carries data over. Each already has a migration test proving the same mapping, and the upgrade rehearsal proves it
against released builds, so they are deleted rather than given an exception to the initiative's rule on stores.

**A revoked session ends its streams.** The login test expires a session mid-stream by deleting its row, because no
public path ends a live stream: a stream checks its session only when it connects. The hub gains that behaviour as a
security property in its own right. Revoking a session closes every stream it holds open.

**UI tests may seed as a runner.** Three board tests produce spend, transcripts, and completions by speaking the fleet
wire as a runner would. That is the real wire, and at `fresh-tenant` it reaches no one else's work, so it stays.

## Work areas

### 1. Scenarios into `tests/service/fleet/`

Each scenario moves at the loosest isolation its reads allow, `fresh-tenant` for any that runs a runner and `fresh-hub`
where its subject is hub-wide, and its Python original is deleted in the same change. Its tick spacing, `poll_until`
intervals, and termination allowance give way to steps and to waits that end when the system has spoken.

### 2. Browser scenarios into `tests/ui/`

The browser scenarios move to `tests/ui/`, the package's one project with a browser installed, against the built board.
Two of them drive the runner's board, and run a real runner host inside the UI project. The pull request gate's UI job
waits on this folder existing.

### 3. The tier retires

`blizzard:e2e` leaves blizzard-context's verifiability matrix when its last file moves, its task, CI job, marker, and
required check leave with it, and the citations of each scenario name the service or UI method that took it. The helpers
the kill-9 crash sweep, the upgrade rehearsal, and the winter tier borrow from `tests/e2e/` move to support modules
first, since those tiers stay in Python.

## Out of scope

The journey rehearsal is `epic:test-suite/journey`'s. The gate's speed is `epic:test-suite/optimization`'s. The web
board's `web:shell-sweep`, a browser test of components with no hub behind them, is neither e2e nor a UI test in the
package's sense, and stays where it is.
