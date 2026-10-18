---
epic: test-architecture
refinement: refined
status: horizon
---

# Plan — `epic:test-architecture`

The e2e, crash-sweep, and journey tiers describe blizzard from the outside, but they reach into its Python to do it. The
goal is for those tiers to drive the platform only through what it exposes: config files, HTTP, CLI, and published
contracts. Then a rewrite of any daemon, in any language, could run them unchanged. The service tier makes the same move
under `initiative:test-suite`, and unit and component tests stay coupled to the Python on purpose; both are out of
scope.

## Current coupling

| Tier            | State            | Coupling                                                                                                                                                                                                                                                                     |
| --------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| E2E             | Hybrid           | Forge, hub, and browser are real processes. The runner loop is ticked in-process in 10 files, and the runner API runs in-process in 6. Config goes through `init_environment` and `RunnerConfig`. 2 files read SQLAlchemy stores. Only 3 files start `blizzard-runner host`. |
| Crash sweep     | Coupled          | The kill is external. Crash points are found by importing 14 Python modules. Leases and tokens are seeded directly into the store. Recovery is asserted through `blizzard.tools.invariants.Invariants` and the store schemas.                                                |
| Journey         | Nearly black-box | Real daemons, HTTP, and CLI. Imports the config dataclasses, `Invariants`, and the crash tier's helpers.                                                                                                                                                                     |
| `blizzard-mock` | Decoupled        | Never imports `blizzard`. Its wire-parity guard reads `wire/facts.py` as source text. `blizzard-mock-data` reflects the SQL schema. No CI of its own.                                                                                                                        |

Four couplings recur across the tiers:

- config built from Python objects;
- state seeded or read through the ORM;
- the runner running in the test process;
- registries (fact kinds, crash points) that exist only as Python source.

## Reference

- **Journey** (`mise run journey`, local only) is a single test that rehearses an overnight run end to end on real
  daemons:
  - five issues are ingested, two of them grouped and one reordered;
  - build → review → deliver, including a multi-repo land, a review-fail loop, an answered question, and an escalation;
  - both daemons are SIGKILLed and restarted mid-run;
  - morning-after assertions: merged to bare `main`, full history, resumed without takeover, nothing worked twice, no
    orphaned environment, and truthful `hub status`.
- **`blizzard-mock`** provides real services over real wires for every seam: a forge on bare git repos, mock harnesses,
  a mock hub, a mock runner, an OAuth IdP, the fixture scaffold, and `blizzard-mock-data`. Each has *levers* that force
  a named misbehaviour.
- **`blizzard-runner tick`** already runs one REAP → PULL → FILL → ADVANCE pass from the CLI (`bzh:steppable-loop`). It
  is a black-box replacement for the in-process `tick_once`.

## Work areas

Area 1 blocks the others, and builds on `epic:test-suite/foundation`.

### 1. External test surface

`initiative:test-suite` publishes most of the surface these tiers need: hub and runner configuration as documented
files, the fact-kind vocabulary under `contracts/`, the runner's host layout as a contract, and a read for each runner
store value its service tests needed. This epic consumes those and adds only what its own tiers still lack, under the
same rule: for each store value read or seeded today, an API or CLI surface, or the SQL schema declared a contract that
tests may query with plain SQL. The ORM is never the contract. `blizzard-mock-data` already seeds both stores from
outside, so a fixture the API cannot create belongs there.

Whether these tiers follow the service tier onto Playwright Test, or stay in pytest behind the guard below, is decided
when this epic comes within striking distance.

### 2. E2E

- `_drive_until_done` calls `blizzard-runner tick --dir <runner>` as a subprocess instead of `tick_once()`. The harness
  environment goes to the child process instead of `os.environ`. Measure the per-tick process cost. If it matters,
  compare it with a `blizzard-runner host` running a short loop interval.
- `_runner_api` starts `blizzard-runner host` instead of serving `build_hosted_app` on a thread.
- Replace the store reads in `test_resume_preamble_e2e.py` and `test_session_modes_e2e.py`, and the imports of
  `harness.preamble`, `events.broker`, and `foundation.chunk_status`, with area 1's surfaces.
- The browser scenarios stay as they are. They already drive the built board through a real hub.

### 3. Crash sweep (investigation)

The arming protocol is already language-neutral: `BLIZZARD_CRASH_POINT=<name>` plus `BLIZZARD_CRASH_FENCE=1` makes the
daemon SIGKILL itself at the named point. Three pieces still need Python:

- **Discovery.** Publish the registry as a committed `contracts/crash-points` artifact with a drift test, or as a
  fence-gated `--list-crash-points` flag on each daemon.
- **Pre-kill state.** Produce leases and tokens through the mock fleet or the API where possible. Record which crash
  windows can only be reached by seeding the store directly.
- **Recovery checks.** Decide whether the invariant checker becomes an operator command (`blizzard-hub check`,
  `blizzard-runner check`) or stays test-only. Identify which recovery assertions can be read through public surfaces.

The output is a decided design and an issue sequence, not the migration itself.

### 4. Journey, `blizzard-mock`, and the guard

- **Journey.** Switch to area 1's config files and area 3's invariant surface. Move the helpers it borrows from the
  crash tier into a shared black-box support module.
- **`blizzard-mock`.** Keep or retire the schema reflection in `blizzard-mock-data` according to area 1's schema
  decision. Add CI: lint, types, tests, and the parity guard against `blizzard` `master`.
- **Guard.** Add an ast-grep or ruff banned-import rule that fails when anything under `tests/e2e/`, `tests/crash/`, or
  `tests/journey/` imports `blizzard.*`. The allowlist shrinks to empty as the areas land.

## Out of scope

The service tier belongs to `initiative:test-suite`, and speed to `epic:test-suite/optimization`. Where speed and
independence conflict on these tiers, the black-box boundary wins.
