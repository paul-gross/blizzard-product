# Landing on the seam

## The `land` action

The actions epic moved `land_pr_ci` into the hub as the `land` action, speaking GitHub through its own client
(`hub/delivery/actions/internal/github_client.py`). This epic deletes that client. `land` reads and acts through the
`RepositoryForge` handed to it for each resolved repository ([seam.md](./seam.md#the-registry)) — its `IForgeRefs`,
`IForgeChanges`, and `IForgeChecks` — and nothing else: `PullRequest` becomes `Change`, `Verdict.of` reads a check's
`outcome` and a change's `mergeability`, and nothing in the action names a GitHub route, field, or credential. Its
stages, outcomes, and the markers it yields are unchanged, and the findings it writes for a failed check still carry the
check's link, now its `details_url`.

The credential is bound inside the adapter the registry built, so the action never handles one. `land` already awaits
the seam on the action thread's loop, so porting it changes how it calls the forge, not where it runs.

`BZ_HUB_GIT_COMMITS` is gone with the actions epic; the action reads the visit's commits, each already resolved to its
repository id.

## What the forge answers

`land` keeps today's answer to every forge refusal it has an answer for, and sends to a person only what no worker can
repair:

| The forge answers                                   | `land` answers                                                                                             |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `ForgeUnavailable`, after the adapter's own retries | `pending`                                                                                                  |
| `ForgeRateLimited(resume_at)`                       | `pending`, its next poll no earlier than `resume_at` through the pass's next-poll hint                     |
| `ForgeRejected` from `merge`                        | Re-reads the change: merged is landed at its `merged_revision`; still open is `pending`, as today          |
| `ForgeRejected` from `open_change`                  | Re-reads the open changes and the comparison: nothing to land, or an open change it adopts, else `pending` |
| `Update(conflict, …)`                               | `conflict`                                                                                                 |
| `Update(refused, …)`                                | `pending`                                                                                                  |
| `ForgeRejected` from reading the closed changes     | `pending`, as an unreadable closed list waits today                                                        |
| `ForgeRejected("auth")`                             | An escalation to a person, cause `forge-auth`                                                              |
| `ForgeNotFound` on the repository                   | An escalation to a person, cause `forge-config`                                                            |
| Any other `ForgeRejected` or `ForgeNotFound`        | `failure`, with the forge's reason in the pass's log                                                       |

A wrong credential or a repository the forge does not know is a fault in the tenant's configuration, which a worker sent
back down `failure` could never repair; the escalation names the repository, its forge, and what the forge said.
`forge-auth` and `forge-config` join `EscalationCause` (`foundation/escalation_causes.py`), the OpenAPI enum, and the
board's cause sentences.

## What an action requires

`IHubAction` gains `requires: frozenset[Capability]`, the capabilities it cannot do without. `land` requires
`refs.read`, `compare`, `changes.open`, `changes.read`, `changes.update-from-base`, `changes.merge.merge`, and
`checks.read`; `record-findings` and `garden-deliver` require nothing.

Before a pass, after repository resolution, `HubNodeExecutor` checks every resolved repository's `offered` capabilities
against the action's `requires`. A repository lacking one refuses the visit: it records a `forge-capability-missing`
event naming the capability, the repository, and its forge, writes a `hub-log.forge-capability-missing` artifact, and
escalates the chunk to a person with cause `forge-capability-missing` before the action runs, since only an operator can
change what a forge offers or what a repository asks of it. The refusal is its own path beside `REFUSAL_EVENT`, which
stays keyed by `RepositoryOutcome`; `forge-capability-missing` joins the closed `EventLogKind`, its severity map
(warning), the census, `EscalationCause`, and `blizzard-context:/domain/operations.md`.

A capability the action can live without is not required. `land` asks `offered` for `checks.rerun` and, without it,
skips its one re-run and routes the failure as it would after one. `garden-deliver` asks it for `commits.read` and,
without it, answers that commit `None`, as a repository it cannot reach answers today.

Because the requirement belongs to the action rather than the graph, a graph names no capability, and every graph
version — old or new — is checked the same way.
