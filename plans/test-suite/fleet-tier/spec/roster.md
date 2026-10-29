# The roster

Each file below is ported in one change that deletes its Python original, keeps each test's function name as its title,
and re-points its citations. "Trace companion" means the file's fleet-trace assertions move to a `fresh-hub` test of
their own beside it ([harness.md](./harness.md#trace-companions)); its core cases keep the level in the table. Times are
the file's share of the last full serial run.

## What moves to `service/fleet/`

| File                                   | Cases    | Level                                                                           | What its port changes                                                                                                                                                                                   | Time  |
| -------------------------------------- | -------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| `test_acceptance_loop.py`              | 5        | `fresh-tenant`, trace companion; tail sampling `fresh-hub`                      | Steps a host under both harnesses. The leak scan reads route tokens as the worker received them ([surfaces.md](./surfaces.md#no-surface-returns-a-secret)). Tail sampling runs against a real collector | 92 s  |
| `test_ask_answer_e2e.py`               | 2        | `fresh-tenant`, trace companion                                                 | Steps exactly as many passes as the original ticked; answers through the `cli` fixture; reads the mock session's resumes                                                                                | 62 s  |
| `test_basic_workspace.py`              | 1        | `fresh-tenant`                                                                  | The host process runs with a `PATH` the test controls, so the winter trap holds ([harness.md](./harness.md#the-runner-host))                                                                            | 24 s  |
| `test_checks_gate_e2e.py`              | 2        | `fresh-tenant`                                                                  | Steps a host                                                                                                                                                                                            | 45 s  |
| `test_delivery_conflict_e2e.py`        | 1        | `fresh-tenant`, trace companion                                                 | Its `merge_conflict` lever targets a repository of the test's own                                                                                                                                       | 20 s  |
| `test_delivery_pr_ci_e2e.py`           | 15 of 16 | `fresh-tenant`; the poll-timeout case's trace companion                         | Steps a host; each lever targets a repository of the test's own; the case that starts a second world and hub takes `fresh-hub`. `…one_lookup_a_pass` moves down ([below](#what-moves-down))             | 387 s |
| `test_escalation_e2e.py`               | 2        | `fresh-tenant`, trace companion                                                 | Runs the takeover command verbatim; reads the mock session's turns and resumes                                                                                                                          | 48 s  |
| `test_event_log_e2e.py`, test 1        | 1        | `fresh-tenant` on the `actions` profile                                         | Waits on the event stream instead of a fixed read window; tests 2 to 4 go to `ui/`, test 5 moves down                                                                                                   | 82 s  |
| `test_flow_record_e2e.py`, test 1      | 1        | `fresh-tenant`                                                                  | Reads flows through the hub ([surfaces.md](./surfaces.md#the-hub)); its invariant checks move down. Test 2 is dropped                                                                                   | 56 s  |
| `test_garden_routine_e2e.py`           | 1        | `fresh-tenant`                                                                  | Finding sets through `GET /api/runs/{id}`, sessions through the chunk-filtered lease read, the graph through its definition read                                                                        | 111 s |
| `test_gate_decision_e2e.py`            | 1        | `fresh-tenant`, trace companion                                                 | Steps a host; decisions through the `cli` fixture                                                                                                                                                       | 21 s  |
| `test_hub_actions_e2e.py`              | 3        | land and test-action cases `fresh-tenant` on `actions`; the refusal `fresh-hub` | The refusal case owns a hub without the fence                                                                                                                                                           | 52 s  |
| `test_ideation_e2e.py`                 | 1        | `fresh-tenant` on `actions`                                                     | Sessions through the chunk-filtered lease read, the graph through its definition read                                                                                                                   | 155 s |
| `test_migration_e2e.py`                | 1        | `fresh-tenant`, trace companion                                                 | The origin's history through the mock forge's history read; its browser half goes to `ui/`                                                                                                              | 25 s  |
| `test_mixed_harness_e2e.py`            | 1        | `fresh-tenant`, trace companion                                                 | Steps instead of polling a timed host; model, effort, harness version, and declared commits through the lease read; its resume-intent checks move down                                                  | 43 s  |
| `test_post_merge_node_e2e.py`          | 1        | `fresh-tenant`                                                                  | Steps a host                                                                                                                                                                                            | 26 s  |
| `test_resume_preamble_e2e.py`          | 2        | `fresh-tenant`                                                                  | Leases through the lease read; the workspace through `GET /api/host/workspaces`; the preamble asserted by its structure, not by the Python's constants                                                  | 71 s  |
| `test_review_cycle_e2e.py`             | 2        | `fresh-tenant`, trace companion                                                 | The companion reads parents and times from the sink ([harness.md](./harness.md#the-sink-and-the-collector))                                                                                             | 72 s  |
| `test_review_findings_delivery_e2e.py` | 1        | `fresh-tenant`                                                                  | The graph through its definition read                                                                                                                                                                   | 39 s  |
| `test_runner_serves_default_e2e.py`    | 1        | `fresh-tenant`                                                                  | `served_projects` through `GET /runners/{id}/api/runner`, against the tenant's own project rather than an assumed `default`                                                                             | 13 s  |
| `test_session_modes_e2e.py`            | 2        | `fresh-tenant`                                                                  | The latest session by its lease's `created_at`, not by a store helper                                                                                                                                   | 68 s  |
| `test_spike_terminal_e2e.py`           | 1        | `fresh-tenant`                                                                  | Its repository is the test's own, so an empty pull list and an unchanged `main` are its alone                                                                                                           | 20 s  |
| `test_egress_events_e2e.py`            | 1        | `fresh-hub` on the `egress` profile                                             | Reads the exported files with a DuckDB reader over `contracts/egress/*.sql`; reships through the `cli` fixture                                                                                          | 70 s  |
| `test_egress_night_e2e.py`             | 10       | `fresh-hub` on `egress`, one hub for the file's night                           | Restarts its hub with a changed format through `freshHub`; its late-usage fold and manifest checks move down                                                                                            | 100 s |
| `test_forge_status_e2e.py`             | 1        | `fresh-hub`                                                                     | Steps the `forge_status` sweep instead of sleeping; reads the skipped sources as an event ([surfaces.md](./surfaces.md#the-hub))                                                                        | 44 s  |

## What moves to `ui/`

| File                                       | Cases | Level                       | What its port changes                                                                                                                               | Time |
| ------------------------------------------ | ----- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| `test_board_browser_e2e.py`                | 1     | `fresh-tenant`              | Steps a host; the landed tree through the mock forge's tree read; dock geometry moves to `web:shell-sweep`                                          | 50 s |
| `test_board_cost_live_e2e.py`              | 1     | `fresh-tenant`              | Seeds spend as a runner over the fleet wire                                                                                                         | 17 s |
| `test_event_log_e2e.py`, tests 2 to 4      | 3     | `fresh-tenant`              | The board's event views, with their baseline counted from the test's own tenant                                                                     | —    |
| `test_gardening_run_dialog_browser_e2e.py` | 1     | `fresh-tenant` on `actions` | The dialog against the test's own project                                                                                                           | 10 s |
| `test_glance_board_e2e.py`                 | 2     | `fresh-tenant`              | The first case steps a host, the second seeds as a runner                                                                                           | 32 s |
| `test_graphs_diagram_browser_e2e.py`       | 3     | `fresh-tenant`              | Packaged graphs through `POST /api/graphs/sync`; the third case's geometry moves to `web:shell-sweep`                                               | 32 s |
| `test_login_session_e2e.py`                | 1     | `fresh-hub`, `auth` profile | Invites and roles through their routes; the session expires by revocation, which closes its stream ([surfaces.md](./surfaces.md#the-hub))           | 22 s |
| `test_migration_e2e.py`, its browser half  | —     | `fresh-tenant`              | Split from the fleet port                                                                                                                           | —    |
| `test_project_lens_browser_e2e.py`         | 1     | `fresh-tenant`              | Unique slugs                                                                                                                                        | 11 s |
| `test_runner_federation_e2e.py`            | 3     | `fresh-hub`, `auth`         | The runner's board on real hosts, laid out through the CLI, including an adopted runner and a host of two ([surfaces.md](./surfaces.md#the-runner)) | 48 s |
| `test_runner_panel_live_e2e.py`            | 1     | `fresh-tenant`              | The runner's board on a timed host                                                                                                                  | 18 s |
| `test_transcript_tab_browser_e2e.py`       | 1     | `fresh-tenant`              | Seeds a transcript as a runner                                                                                                                      | 10 s |
| `test_triage_history_browser_e2e.py`       | 1     | `fresh-tenant`              | Syncs graphs and seeds a completion as a runner                                                                                                     | 12 s |

## What moves down

A test whose subject is internal machinery moves to the component tier, into the `tests/` file that owns that machinery,
marked `component`:

- `test_delivery_pr_ci_e2e.py`'s `…one_lookup_a_pass`, which proves the forge client's conditional reads by scraping the
  mock forge's request log.
- `test_event_log_e2e.py`'s test 5, which proves how the hub's event broker ring meets a backfill across a restart.
- `test_egress_night_e2e.py`'s late-usage fold and manifest digest checks, which inject a usage fact on a sequence line
  a live host owns.
- `test_mixed_harness_e2e.py`'s resume-intent set and clear.
- `test_flow_record_e2e.py`'s invariant checks, beside the checker's own tests.

## What is dropped

`test_forge_carry_over_e2e.py`, `test_run_carry_over_e2e.py`, and `test_flow_record_e2e.py`'s test 2 migrate a hub down,
seed or read its stopped store, and migrate it back up. The migrations each proves are already proved by
`20261025_0900_forges_test.py` and `20261023_0901_hub_actions_test.py` under `src/blizzard/hub/store/migrations/tests/`,
and against released builds by the upgrade rehearsal. Each is deleted in a change of its own that names the test which
covers it, and `tests/test_run_step_census.py`'s allowlist entry for `run_carry_over` leaves with it.
