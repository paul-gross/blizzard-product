---
epic: test-suite/fleet-tier
refinement: scaffolded
status: horizon
---

# Plan — `epic:test-suite/fleet-tier`

Today's e2e tier is misnamed. Every scenario in it runs against the mock forge and mock harnesses, never a real work
source or a real GitHub repository, so it is the hub and a runner host held real with mocks around them, which is what
the initiative calls a fleet service test. Its browser scenarios are UI tests: the built board in a browser, driving a
real hub. Each one also starts a forge and a hub of its own, and the tier runs them one at a time, so a full run costs
about thirty-five minutes on a hosted machine. This epic moves each scenario to where its kind lives in the Playwright
package, and retires the e2e tier until blizzard can stand up a real work source under test.

It follows the runner tier because it needs what that epic builds: a `blizzard-runner host` that steps on request and
serves its own local API, configuration as documented files, and reads for the runner state its tests check.

## The tests it moves

| Coupling today                              | Where                                                                            | What replaces it                                                        |
| ------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| The runner loop ticked in-process           | about 20 files, through `drive_until_done` and `tick_once`                       | the runner tier's stepping host, stepped through its local API          |
| The runner API served on a thread           | about 10 files, through `runner_api` and `build_hosted_app`                      | the same host serving that API itself                                   |
| Configuration built from Python objects     | `RunnerConfig`, `init_environment`, the hub and tracing configs                  | the documented configuration files                                      |
| The harness environment set on `os.environ` | the scenarios that pick a harness                                                | the environment of the host process                                     |
| The store read through the ORM              | `test_resume_preamble_e2e.py`, `test_session_modes_e2e.py`, and one other        | a public read of each value, built here where the runner tier left none |
| Internal machinery imported as an oracle    | `harness.preamble`, `events.broker`, `foundation.chunk_status`, and a few others | the behaviour those modules produce, observed through public surfaces   |

## Work areas

### 1. Scenarios into `tests/service/fleet/`

Each scenario moves at the loosest isolation its reads allow: `fresh-tenant` for any that runs a runner, `fresh-hub`
where its subject is hub-wide. Its Python original is deleted in the same change, and its tick spacing, `poll_until`
intervals, and termination allowance give way to waits that end when the system has spoken.

### 2. Browser scenarios into `tests/ui/`

The `*_browser_e2e.py` files and the shell sweep move to `tests/ui/`, the package's one project with a browser
installed, against the frontend job's build. The pull request gate's UI job waits on this folder existing.

### 3. The tier retires

`blizzard:e2e` leaves blizzard-context's verifiability matrix when its last file moves, `mise run e2e` and the
`upper-tiers.yml` job leave with it, and the citations of each scenario name the service or UI method that took it.

## Out of scope

The journey rehearsal is `epic:test-suite/journey`', and the kill-9 crash sweep stays in Python, keeping whatever it
borrows from `tests/fleet_launchers.py` in a Python support module. The gate's speed is
`epic:test-suite/optimization`'s.
