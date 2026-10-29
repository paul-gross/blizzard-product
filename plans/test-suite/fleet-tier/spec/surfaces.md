# Surfaces

## The decision

Every value an e2e test reads today through a store or an imported module is read through a public surface an operator
or an integration could use as well, built in the change of the first port that needs it, secured like the routes beside
it, and noted in the OpenAPI document and a change note under `changes/unreleased/`. What the runner tier's
[reads](../../runner-tier/spec/reads.md) already serve is used as it stands. A read built here is additive to every
supported release.

## No surface returns a secret

Three files scan transcripts, spans, and artifacts for leaked route tokens, and read the tokens from the runner's
`route_tokens` table to know what to look for. No route serves a token. The scan reads each route token as the worker
received it, from the spawn environment the mock harness records for each session, as the harness's `recordedLeaseToken`
already reads a lease token. A token that never reaches a worker is planted by the scan's own sentinel, as
`PLANT_LEASE_TOKEN_SCRIPT` plants one today.

## The hub

| Read today                                                     | In                                | Read instead                                                                                                                              |
| -------------------------------------------------------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `flow_submissions` and `flow_submission_closures`, hub running | `test_flow_record_e2e.py`         | A chunk's flow submissions and their closures through the chunk's API, tenant-bound                                                       |
| `finding_sets`, hub running                                    | `test_garden_routine_e2e.py`      | `GET /api/runs/{id}`'s `sets`, which already serves them                                                                                  |
| A packaged graph's definition, through `PACKAGED.named`        | garden, ideation, review findings | `GET /api/graphs/{name}` gains the graph's definition as published, so a reader sees the nodes and prompts a run follows                  |
| The hub's event kinds, through `HubEventType`                  | `test_event_log_e2e.py`           | The event log's own `kind` field, against the kinds the OpenAPI document names                                                            |
| "Sources skipped", an untagged line in `daemon.log`            | `test_forge_status_e2e.py`        | An entry on the hub's event log when a forge-status pass skips a source it cannot reach                                                   |
| A session expired by deleting its row                          | `test_login_session_e2e.py`       | Revoking a session closes every stream it holds open, so expiry is observed by logging out ([below](#a-revoked-session-ends-its-streams)) |
| Memberships, graphs, and chunks inserted, `invite()` called    | `test_login_session_e2e.py`       | The invite, role, graph, and chunk routes that already exist                                                                              |

The board's reads the UI tests rely on, `GET /api/events`, `/api/activity`, `/api/garden-proposals`, `/api/decisions`,
and `POST /api/graphs/sync`, are each confirmed tenant-bound in the change of the first port that reads it on a shared
hub, and made so where it is not. A test that asserts an exact list reads only its own tenant's.

### A revoked session ends its streams

A stream checks its session when it connects, so a session revoked while a board holds a stream open keeps receiving
until the stream ends on its own. The hub closes each open stream of a session when that session is revoked, by logout
or by an administrator, and a client reconnecting with it is refused as any request would be. This is a property the hub
owes its users with or without a test, and the login port is the test that holds it to it.

## The runner

| Read today                                                                                               | In                                                       | Read instead                                                                                                                                      |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| A chunk's sessions, through leases in the store                                                          | garden, ideation, session modes, resume preamble         | `GET /runners/{id}/api/leases?chunk_id=…`, every lease of that chunk, open or closed, unbounded by the recent-lease cap of twenty                 |
| A lease's resolved model and effort, `leases ⋈ lease_context`                                            | `test_mixed_harness_e2e.py`, `test_session_modes_e2e.py` | The lease read gains `resolved_model` and `resolved_effort`                                                                                       |
| A usage fact's harness version                                                                           | `test_mixed_harness_e2e.py`                              | The lease read gains `harness_version`                                                                                                            |
| A lease's git-commit declarations                                                                        | `test_mixed_harness_e2e.py`                              | The lease read gains the commits its worker declared                                                                                              |
| The latest session, through `store.latest_session`                                                       | `test_session_modes_e2e.py`                              | The chunk-filtered lease read, ordered by `created_at`                                                                                            |
| `served_projects_answer()` from the stores                                                               | `test_runner_serves_default_e2e.py`                      | `GET /runners/{id}/api/runner`'s `served_projects`, which already serves it                                                                       |
| The runner's name, through `load_runner_config`                                                          | escalation takeover                                      | `GET /api/host/runners`                                                                                                                           |
| An adopted runner and a host of two, through `HostLayout`, `RunnerEntry`, `HubTokenFile`, `adopt_runner` | `test_runner_federation_e2e.py`                          | `blizzard-runner init`, `add`, and adoption through the CLI. Where the CLI cannot adopt a runner on a hub that signs people in, it gains the verb |

## The mock fleet

| Need                                             | In                                       | Surface                                                                                                |
| ------------------------------------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| A repository's tree and history                  | board, migration, spike terminal         | `blizzard-mock-forge` gains reads of a repository's tree at a ref and of its commit history            |
| A lever scoped to one test                       | every file that pulls a repository lever | The harness gives each test a repository of its own, so a repository-keyed lever reaches no other test |
| A mock session's turns, resumes, and invocations | ask, escalation, session modes, mixed    | The harness's `mockSessions()`, which reads what the mock harness records                              |

The forge's request log stays unread: the one test that scraped it moves down.
