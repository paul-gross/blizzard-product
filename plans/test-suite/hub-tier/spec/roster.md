# The roster

## What moves

Each file below becomes one spec in `tests/service/hub/`, or in the profile subfolder named, its tests keeping their
function names as titles. The level is where classification expects the file to land, the loosest its reads look to
allow. The read rules place each test, and a test whose reads allow a looser level than the table expects lands there. A
file whose tests need different levels sets the level per `describe` block.

| File                                            | Tests | Profile                                                    | Expected level                   | What its port changes                                                                                                                                                                                                                                                                                                                                                                |
| ----------------------------------------------- | ----- | ---------------------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `test_hub_service.py`                           | 25    | default, `route-token-enforce` (3), `produces-enforce` (5) | `common` to `fresh-tenant`       | The enforcement tests move into their profiles. `test_sigterm_returns_promptly_with_a_client_parked_on_the_stream` and `test_a_fresh_migration_publishes_queue_changed_to_a_live_subscriber` take `fresh-hub`. Its two `collect(window=6.0)` waits are rewritten ([waits.md](./waits.md)). `_graph_yaml` moves out first ([carry-over.md](./carry-over.md#helpers-that-move-first)). |
| `test_finding_exits_service.py`                 | 11    | default                                                    | `fresh-tenant`                   | A mock runner per test, through `mockRunner`. The `blizzard` CLI runs through `cli` ([harness.md](./harness.md#the-cli)), naming its tenant by `BZ_HUB_TENANT`.                                                                                                                                                                                                                      |
| `test_event_log_service.py`                     | 4     | default                                                    | `fresh-hub`                      | The hub event log belongs to no tenant. The test reading `trace-config-rejected` rows through `tests/hub_event_rows.py` reads them through `GET /api/admin/events` instead. Its `extra_env` OTEL variables become the fresh hub's own environment ([harness.md](./harness.md#a-fresh-hubs-own-configuration)). Its `collect(window=6.0)` wait is rewritten.                          |
| `test_runner_served_projects_service.py`        | 4     | default                                                    | `fresh-tenant`                   | Runners are tenant-wide.                                                                                                                                                                                                                                                                                                                                                             |
| `test_transcript_segments_service.py`           | 4     | default                                                    | `fresh-tenant`                   | Fleet calls under a runner token, through `fleet`. Its records are typed literals from the wire types, in place of `transcript_segment_record`.                                                                                                                                                                                                                                      |
| `test_projects_ingest_delivery_service.py`      | 3     | default                                                    | `fresh-tenant`                   | A mock runner per test.                                                                                                                                                                                                                                                                                                                                                              |
| `test_routine_baselines_service.py`             | 3     | default                                                    | `fresh-tenant`                   | `garden_stack`'s forge, hub, and mock runner become the shared hub's `tenant`, `repository`, and `mockRunner`.                                                                                                                                                                                                                                                                       |
| `test_forge_status_service.py`                  | 2     | default                                                    | `fresh-tenant`                   | Each test steps the forge-status sweep over its own tenant with `api.sweep` instead of waiting on a one-second interval. `test_a_non_opted_in_source_starts_no_sweep_loop` asserts after one completed pass instead of three seconds ([waits.md](./waits.md#the-forge-status-model-case)).                                                                                           |
| `test_runner_declining_service.py`              | 2     | default                                                    | `fresh-tenant`                   | Runners are tenant-wide.                                                                                                                                                                                                                                                                                                                                                             |
| `test_work_source_records_service.py`           | 2     | default                                                    | `fresh-tenant`                   | The test that sets `annotation_interval_seconds=1` steps the forge-status sweep instead.                                                                                                                                                                                                                                                                                             |
| `test_usage_service.py`, two of three           | 2     | default                                                    | `fresh-tenant`                   | The two that drive a real hub with a mock runner. The third, against the mock hub, stays for the runner tier.                                                                                                                                                                                                                                                                        |
| `test_platform_tracing_service.py`, two of five | 2     | default                                                    | `fresh-hub`                      | `test_a_hub_with_platform_on_delivers_request_and_query_spans` and `test_a_hub_with_platform_unset_delivers_no_platform_span`. Tracing export is hub-wide. Each configures its fresh hub's tracing and exports to the harness's OTLP sink ([harness.md](./harness.md#the-otlp-sink)), and stops the hub to flush. The other three start a runner host and wait for the runner tier.  |
| `test_runner_auth_service.py`, one of three     | 1     | default                                                    | `fresh-tenant`                   | `test_a_running_hub_refuses_a_revoked_token_and_a_tokenless_call`. The other two stay for the runner tier.                                                                                                                                                                                                                                                                           |
| `test_auth_login_service.py`                    | 7     | `auth`                                                     | `fresh-tenant`, some `fresh-hub` | People through `user` and `people` ([harness.md](./harness.md#people)), never `seed_invitation`. Tests that vary the providers or depend on the superuser bootstrap take `fresh-hub` with a configuration of their own. `invite` moves out first.                                                                                                                                    |
| `test_cli_login_service.py`                     | 3     | `auth`                                                     | `fresh-tenant`                   | The CLI's login runs through `cli` with the person's IdP identity registered beforehand.                                                                                                                                                                                                                                                                                             |
| `test_event_stream_revocation_service.py`       | 1     | `auth`                                                     | `fresh-tenant`                   | Its `_MemberStream` becomes `events` opened in the member's session. The one-second drain is dropped, since the stream is opened before the act ([waits.md](./waits.md#opening-the-stream)).                                                                                                                                                                                         |

The profiles are subfolders of `tests/service/hub/`, each with its `profile.toml`:

| Profile               | What it sets                                                                                                                          |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `route-token-enforce` | `route_token_mode` as `ROUTE_TOKEN_ENFORCE` names it                                                                                  |
| `produces-enforce`    | `produces_mode` as `PRODUCES_ENFORCE` names it                                                                                        |
| `auth`                | Authentication on, the stub IdP as the one OIDC provider, a superuser, and the login throttle off ([harness.md](./harness.md#people)) |

A test that uses one of these constants in its body takes the value from the wire types or restates it as a literal,
since a spec cannot import `blizzard`.

## What stays in Python

These files stay in the service tier's pytest half, unchanged, because their subject is Python itself:

| File                                     | Its subject                                         |
| ---------------------------------------- | --------------------------------------------------- |
| `test_opencode_compatibility_service.py` | The OpenCode adapter driven against `mock-opencode` |
| `test_harness_health_service.py`         | Harness health checks against the mock binaries     |
| `test_credential_renewal_service.py`     | Credential renewal against `mock-codex app-server`  |
| `test_runner_selftest_mock_service.py`   | The runner's self-test against `mock-claude-code`   |
| `test_wire_parity_service.py`            | The Python hub client against the mock hub          |
| `test_parity_guard.py`                   | The Python client's protocols against the mocks     |
| `test_mock_data_seeding_service.py`      | `blizzard-mock-data` writing into a hub's store     |
| `test_host_token_service.py`             | The mock hub's host tokens                          |
| `test_runner_bindings_service.py`        | The mock hub's runner bindings                      |
| `test_mock_chunk_findings_service.py`    | The mock hub's chunk findings                       |

`test_tenant_teardown_service.py` builds its hub in-process with `build_hub` and calls
`hub_services.tenant_teardown.delete` directly, so it is a component test in the service folder. It moves to
`tests/test_tenant_teardown.py`, marked `component`, unchanged otherwise.

## What waits for the runner tier

Every other file in `tests/service/` starts a runner in-process or as a host, against the mock hub or beside a real one,
and is `epic:test-suite/runner-tier`'s. That includes `test_runner_init_service.py` and
`test_idp_federation_service.py`, the two that start a real runner host beside a real hub and land in
`tests/service/fleet/`, and the runner halves of the three split files above.
