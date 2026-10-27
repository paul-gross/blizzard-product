# The roster

## What moves to `service/runner/`

Each test below drives a host of its own against a mock hub of its own, so it declares no isolation level. A file that
runs its runner in-process today steps a stepped host instead; one that starts a host today keeps doing so, on the
harness's layout.

| File                                                                                                                                             | Tests  | Coupling today                                         | What its port changes                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `test_runner_service.py`                                                                                                                         | 24     | in-process and host, store reads, levers               | Steps a host and reads the API. Its three stream tests: one ports with waits rewritten; the two that publish into an in-process `EventBroker` move down ([below](#what-moves-down)). |
| `test_host_halted_runner_service.py`                                                                                                             | 8      | host, `RunnerEntry`, store, levers                     | Halted runners laid out by the contract's files                                                                                                                                      |
| `test_mixed_harness_dispatch_service.py`                                                                                                         | 8      | in-process, calls `Fill` and `Pull`                    | Steps a host; the cases that call a step class directly move down                                                                                                                    |
| `test_opencode_service.py`                                                                                                                       | 7      | in-process, store                                      | Steps a host with `mock-opencode`                                                                                                                                                    |
| `test_host_directory_owner_service.py`                                                                                                           | 5      | host, a lock-holder subprocess, `sqlite3`              | Into `runner/store/`; the lock is held by a second real host rather than by `HostLock.acquire`, and the legacy `runner.sock` listener is the harness's own                           |
| `test_runner_garden_proposals_service.py`                                                                                                        | 3      | `runner_api`, config                                   | Steps a host                                                                                                                                                                         |
| `test_runner_auth_service.py`, two of three                                                                                                      | 2      | in-process, store                                      | Steps a host; the third moved with the hub tier                                                                                                                                      |
| `test_system_artifacts_service.py`                                                                                                               | 3      | `runner_api`, CLI, store seeding                       | Steps a host; the CLI through `cli`                                                                                                                                                  |
| `test_platform_tracing_service.py`, three of five                                                                                                | 3      | host, store                                            | Tracing in the runner profile, exported to the harness's OTLP sink                                                                                                                   |
| `test_usage_service.py`, one of three                                                                                                            | 1      | in-process, store, levers                              | Steps a host                                                                                                                                                                         |
| `test_spend_figure_service.py`, `test_runner_analytics_service.py`, `test_runner_finding_service.py`, `test_runner_garden_findings_service.py`   | 2 each | in-process, `runner_api`                               | Step a host. Their docstrings' claim of a host subprocess becomes true                                                                                                               |
| `test_work_items_service.py`                                                                                                                     | 2      | `runner_api`, `pin_runner`                             | Steps a host                                                                                                                                                                         |
| `test_host_init_readd_service.py`, `test_host_registration_service.py`                                                                           | 2 each | host, store                                            | Read through the API                                                                                                                                                                 |
| `test_dashboard_route_service.py`, `test_claude_code_bundle_service.py`, `test_subscription_floor_service.py`, `test_runner_selector_service.py` | 1 each | various                                                | Step a host or keep a timed one; read through the API                                                                                                                                |
| `test_utilization_lever_service.py`                                                                                                              | 1      | host, live sample inserts, fixed worker delay          | Step real sampling against a local vendor counterpart; use a worker checkpoint and public lease/decline reads                                                                        |
| `test_budget_lever_service.py`                                                                                                                   | 2      | host, live spend and sample inserts                    | Accrue actual priced mock-worker spend and sample through the configured vendor counterpart; keep budget expiry distinct from a latched ceiling                                      |
| `test_release_service.py`                                                                                                                        | 3      | multi-runner host, live sample inserts and store reads | Drive sampled admission and real workers; observe release, cold reclaim, usage accounting and held state through public reads                                                        |

## What moves to `service/fleet/`

| File                             | Tests | Level                                                                                                                             |
| -------------------------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------- |
| `test_runner_init_service.py`    | 5     | `fresh-tenant`; `test_after_its_hub_s_data_is_reset_…` takes `fresh-hub` and restarts its hub on the same port through `freshHub` |
| `test_idp_federation_service.py` | 3     | `fresh-hub`, on the `auth` profile's folder                                                                                       |

## What moves down

A test whose subject is the loop's internal machinery moves to the component tier, into the `tests/` file that owns that
machinery, marked `component`:

- `test_runner_stream_resumes_live_after_a_restart_reset_the_broker_ids` and
  `test_runner_stream_replays_a_restarted_brokers_buffered_tail_past_a_stale_cursor`, which publish frames into an
  `EventBroker` directly to prove how a restarted broker numbers them.
- The cases of `test_mixed_harness_dispatch_service.py` that call `Fill` or `Pull` themselves.

## What stays in the pytest half

`test_host_graduation_service.py` and `test_host_ingest_service.py` seed legacy and second stores through SQLAlchemy's
`reflect` and run `blizzard-runner migrate` and `ingest`. Their subject is the store's files, and a port would restate
each legacy schema in SQL for no gain in independence, since the files are the contract either way. They stay in the
pytest half, with their reads moved to the contract's names. `test_wire_parity_service.py` stays, as
[the hub tier](../../hub-tier/spec/roster.md#what-stays-in-python) lists.
