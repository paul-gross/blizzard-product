# Surfaces

## The decision

Every value an e2e test reads today through a store or an imported module is read through a public surface an operator
or an integration could use as well, built in the change of the first port that needs it, secured like the routes beside
it, and noted in the OpenAPI document and a change note under `changes/unreleased/`. What the runner tier's
[reads](../../runner-tier/spec/reads.md) already serve is used as it stands. A read built here is additive to every
supported release.

## No surface returns a secret

Three files scan transcripts, spans, and artifacts for leaked route tokens, reading the tokens from the runner's
`route_tokens` table to know what to look for. A route token is minted by the hub in the claim and kept by the runner;
it never reaches a worker, and it carries no prefix a scan could match. No route will serve one, so the check that route
tokens never leave the runner moves down to a component test of span and artifact assembly
([roster.md](./roster.md#what-moves-down)). The trace companions keep the scans an outside reader can make: the lease
token, read as the worker received it through the harness's `recordedLeaseToken`, the runner's bearer, and the planted
artifact body.

## The hub

| Read today                                                     | In                                | Read instead                                                                                                                                                             |
| -------------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `flow_submissions` and `flow_submission_closures`, hub running | `test_flow_record_e2e.py`         | A chunk's flow submissions and their closures through the chunk's API, tenant-bound                                                                                      |
| `finding_sets`, hub running                                    | `test_garden_routine_e2e.py`      | `GET /api/runs/{id}`'s `sets`, which already serves them                                                                                                                 |
| A packaged graph's authoring body, through `PACKAGED.named`    | garden, ideation, review findings | `GET /api/graphs/{graph_id}` already serves the reified definition, prompts included; it gains the graph's authoring body as published, which a test edits and resubmits |
| The hub's SSE event types, through `HubEventType`              | `test_event_log_e2e.py`           | The event types `contracts/sse/` documents                                                                                                                               |
| "Sources skipped", an untagged line in `daemon.log`            | `test_forge_status_e2e.py`        | An entry on the hub's event log when a forge-status pass skips a source it cannot reach                                                                                  |
| A session ended by deleting its row                            | `test_login_session_e2e.py`       | Logging out ends the session's open streams ([below](#logging-out-ends-a-sessions-streams))                                                                              |
| Memberships, graphs, and chunks inserted, `invite()` called    | `test_login_session_e2e.py`       | The invite, role, graph, and chunk routes that already exist                                                                                                             |

The board's reads the UI tests rely on, `GET /api/events`, `/api/activity`, `/api/garden-proposals`, `/api/decisions`,
and `POST /api/graphs/sync`, are each confirmed tenant-bound in the change of the first port that reads it on a shared
hub, and made so where it is not. A test that asserts an exact list reads only its own tenant's.

### Logging out ends a session's streams

The hub's event stream already ends when the member it serves loses their membership, and the hub tier holds it to that.
A session that logs out does not end its streams: a stream checks its session when it connects, and runs on after the
session is gone. The hub closes each open stream of a session when that session logs out, and a client reconnecting with
it is refused as any request would be. The change lands with component tests beside the membership case, ahead of the
login port, which observes a session's end through it.

## The runner

| Read today                                                                                               | In                                                       | Read instead                                                                                                                      |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| A chunk's sessions, through leases in the store                                                          | garden, ideation, session modes, resume preamble         | `GET /runners/{id}/api/leases?chunk_id=…`, every lease of that chunk, open or closed, unbounded by the recent-lease cap of twenty |
| A lease's resolved model and effort, `leases ⋈ lease_context`                                            | `test_mixed_harness_e2e.py`, `test_session_modes_e2e.py` | The lease read gains `resolved_model` and `resolved_effort`                                                                       |
| A usage fact's harness version                                                                           | `test_mixed_harness_e2e.py`                              | The lease read gains `harness_version`                                                                                            |
| A lease's git-commit declarations                                                                        | `test_mixed_harness_e2e.py`                              | The lease read gains the commits its worker declared                                                                              |
| The latest session, through `store.latest_session`                                                       | `test_session_modes_e2e.py`                              | The chunk-filtered lease read, ordered by `created_at`                                                                            |
| `served_projects_answer()` from the stores                                                               | `test_runner_serves_default_e2e.py`                      | `GET /runners/{id}/api/runner`'s `served_projects`, which already serves it                                                       |
| The runner's name, through `load_runner_config`                                                          | escalation takeover                                      | `GET /api/host/runners`                                                                                                           |
| An adopted runner and a host of two, through `HostLayout`, `RunnerEntry`, `HubTokenFile`, `adopt_runner` | `test_runner_federation_e2e.py`                          | The host files the host contract documents, written by the harness as the runner tier's layout does; no new CLI verb              |

## The mock fleet

| Need                                             | In                                       | Surface                                                                                                                                 |
| ------------------------------------------------ | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| A repository's tree, history, and file contents  | board, migration, spike terminal         | The harness's reads of the bare origin, beside `originHead` ([harness.md](./harness.md#the-runner-host)). `blizzard-mock` gains nothing |
| A lever scoped to one test                       | every file that pulls a repository lever | The harness gives each test a repository of its own, so a repository-keyed lever reaches no other test                                  |
| A mock session's turns, resumes, and invocations | ask, escalation, session modes, mixed    | The harness's `mockSessions()`, which reads what the mock harness records                                                               |

The forge's request log stays unread: the one test that scraped it moves down.
