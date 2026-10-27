# Reading state

## The decision

Every value a runner test reads today through the ORM is read through the runner's local API or its CLI, where an
operator would want to read it too. A test reads the store by plain SQL only when the store itself is its subject,
against a host that is stopped, by the names `contracts/store/runner.json` pins. The ORM is never the contract, and no
test reads a running host's store.

| Value read today                                       | Through                                 | Read instead                                                                                                                                                                                   |
| ------------------------------------------------------ | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Leases, open and closed                                | `SqlAlchemyRunnerStore`, `daemon_store` | `GET /runners/{id}/api/leases`                                                                                                                                                                 |
| Escalations and their closures                         | the `escalation_closures` table         | `GET /runners/{id}/api/escalations`, which shows an escalation's closure ([below](#what-the-api-gains))                                                                                        |
| Pending outbound facts                                 | the `outbound_buffer` table             | `GET /runners/{id}/api/runner`'s `hub.buffer_depth` for how many; `GET /runners/{id}/api/facts`, each fact's `acked_at` null while it is pending                                               |
| Usage samples                                          | `external_usage_samples`                | `GET /api/host/subscriptions`, each subscription's latest sample                                                                                                                               |
| A local pause                                          | `local_pause_facts`                     | `GET /runners/{id}/api/runner`'s `pause.local` and `local_reason`                                                                                                                              |
| The runner's identity                                  | `make_store`, `runner_identity`         | `GET /runners/{id}/api/runner`'s identity                                                                                                                                                      |
| A lease's token, for a worker call a test makes itself | the `leases` row                        | The test acts as the worker would: it reads the token from the spawn environment the mock harness records for each session, a recording `blizzard-mock` gains if the spike finds it keeps none |
| Store files across a graduation, an ingest, or a lock  | `reflect`, `rows`, `sqlite3`            | Plain SQL by the contract, the host stopped ([below](#plain-sql))                                                                                                                              |

`/api/facts` is history, not a queue (`bzh:facts-not-status`), and a test reading a fact's `acked_at` reads that fact's
history, which is what it asserts on; a test that needs the queue's depth reads the count.

A test whose subject is delivery of one particular fact reads both: queue depth establishes the backlog, while the facts
feed identifies the test's own `kind`, `chunk_id`, and `seq` and follows that sequence's `acked_at` through failure and
recovery. In the unreachable-hub case, the fact is `completion.submitted`. A positive depth alone is insufficient: an
earlier fact can be queued before a worker has completed at all. The harness client exposes these existing feed fields;
no new API read is needed for this case.

## What the API gains

The table names only reads that exist today, save one: `GET /runners/{id}/api/escalations` gains each escalation's
closure, `closed_at` and how it closed, if the spike or the port finds it does not show them. Any further value a port
proves missing is added to the local API in the change that ports its test, under the plan's order of preference, and
named here.

## Plain SQL

A test whose subject is the store, the graduation of a legacy `data/runner.db`, an ingest of one store into another, or
the ownership of a host directory, may write files before the host starts and read them after it stops, with Node's own
`node:sqlite`, by the table and column names `contracts/store/runner.json` pins. Such tests live in
`tests/service/runner/store/`, the one folder whose specs the package's lint allows to import `node:sqlite` and
`node:fs`. A schema the contract does not pin, such as the legacy `runner.db` a graduation starts from, is written from
SQL kept beside the test, copied from the migration that produced it.
