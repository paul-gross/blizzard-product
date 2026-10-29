# Retirement

## The support modules first

`tests/e2e/` is imported by tiers that stay in Python, so nothing in it is deleted until what they borrow has moved:

| Importer                                                                                                      | Borrows                                                                                                                                                  | Moves to                                                |
| ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `tests/fleet_world.py`, and through it the crash sweep and the journey                                        | `tests/e2e/fleet_traces.py`'s `PLANT_LEASE_TOKEN_SCRIPT` and `SENTINEL_ARTIFACT_BODY`; `tests/e2e/harness_variants.py`'s `CLAUDE_CODE` and `MockHarness` | `tests/fleet_traces.py` and `tests/harness_variants.py` |
| `tests/fleet_launchers.py`, and through it the rehearsal, the winter tier, and the service tier's pytest half | `tests/e2e/fleet_traces.py`'s `PLANT_DIR_VAR` and `FleetCollector`, behind `run_hub(collector=…)`                                                        | `tests/fleet_traces.py`                                 |
| `tests/crash/test_basic_workspace_crash.py`                                                                   | `tests/e2e/test_basic_workspace.py`'s `bare_origin`, `basic_config`, `basic_graph_yaml`, and `trap_winter`                                               | `tests/fleet_world.py`                                  |

Each move re-points every importer in the same change and changes no assertion. Helpers shared only among e2e files,
such as `_tick_until`, `_drive_until`, `_graph_yaml`, and `_sessions_by_node`, are not moved: they leave with the last
file that uses them.

## What names the tier

When the last file has moved, one change removes every place the tier is named:

**`blizzard`**

- `mise.toml`'s `[tasks.e2e]`. Its `otelcol-contrib` pin stays in `[tasks.collector-config-check]`, and the UI task
  takes its dependency on `web-build`. The `gate-full` description and `scripts/gate-full.sh`'s tier list drop it.
- `.github/workflows/upper-tiers.yml`'s `e2e` job and header, and the comments naming it in `pr.yml`, `push.yml`,
  `release.yml`, and `gate.yml`.
- The required check `upper-tiers / e2e acceptance loop (blizzard:e2e)`, in branch protection and in blizzard-context's
  `verification/blizzard/commands/packaging.md`.
- `docs/ci.md`, including its claim that e2e is the slowest job.
- `pyproject.toml`'s `e2e` marker and its `playwright` dev dependency, which only `tests/e2e/` uses, and `uv.lock`.
- The tests that pin the tier: `tests/test_upper_tiers_workflow.py`, `tests/test_release_workflow.py`,
  `tests/test_winter_free_ci.py`, `tests/suite_plugin.py`, `tests/test_default_suite_markers.py`, and
  `tests/test_config_keys_reach_a_gating_tier.py`.
- `scripts/prose-density-baseline.json`'s e2e entries.

**`blizzard-context`**

- `verification/blizzard.md`'s `blizzard:e2e` method. `blizzard-mock:e2e`, the mock fleet's own tier, stays.
- `verification/blizzard/e2e-scenarios.md` and its nine spokes. Each scenario's citation names the service or UI test
  that took it, and the scenarios fold into the service and UI method pages.
- `verification/blizzard/commands.md`, `tier-rules.md`, and every page citing the method: `gaps.md`, `manual.md`,
  `pre-push.md`, `evidence.md`, `acceptance-criteria.md`, `companion-changes.md`, `tools.md`, and the command pages for
  the journey, the service tier, the crash sweep, the winter binding, mutation, and packaging.
- `scripts/check-registry-drift.py`'s e2e marker, method pattern, tier inference, and its e2e roster completeness check,
  with their tests, and `scripts/registry-copies.json`'s `e2e-scenarios` entry.
